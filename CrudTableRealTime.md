/**
 * CrudTable.RealTime.js
 * ─────────────────────────────────────────────────────────────
 * Módulo de colaboración en tiempo real para CrudTable.
 * 
 * Se encarga de:
 *  • Conectar al WebSocket del canal correspondiente a la tabla.
 *  • Avisar a otros usuarios cuando esta tabla edita o agrega un registro.
 *  • Mostrar visualmente quién está trabajando con cada registro.
 *  • Refrescar la tabla automáticamente cuando otro usuario guarda cambios.
 * 
 * Uso:
 *   const rt = new CrudTableRealTime(crudTableInstance);
 *   rt.start();
 * 
 * El módulo se acopla a CrudTable mediante:
 *   - crudTable.config.wsChannel      (nombre del canal)
 *   - crudTable.config.tableId        (id del <table> en el DOM)
 *   - crudTable._refresh()            (para recargar datos)
 *   - window.__currentUser            (nombre del usuario actual)
 */

(function () {
    'use strict';

    // Paleta de colores disponibles para usuarios activos.
    // Cada usuario recibe un color determinista según su nombre
    // (mismo usuario = mismo color, siempre).
    const USER_COLOR_PALETTE = [
        '#d24e74', '#8a0a08', '#04ed1c', '#778df2',
        '#000000', '#122a71', '#FF3838', '#FFB302',
        '#56F000', '#2DCCFF', '#A4ABB6', '#133306',
        '#4f60b3', '#0d7100', '#6c5f97', '#1effbc',
        '#ecd651'
    ];

    // Cuánto tiempo espera el cliente antes de intentar reconectar,
    // y cada cuánto hace la verificación periódica para mantener los indicadores visibles.
    const RECONNECT_BASE_MS = 1000;
    const RECONNECT_MAX_MS = 30000;
    const RECONNECT_MAX_ATTEMPTS = 20;
    const WATCHDOG_INTERVAL_MS = 1500;

    class CrudTableRealTime {
        constructor(crudTable) {
            this.crud = crudTable;
            this.me = (window.__currentUser || '').trim();

            this.socket = null;
            this.reconnectAttempts = 0;

            // Estado de filas bloqueadas por otros usuarios: id -> {user, color}
            this.rowsBeingEdited = new Map();
            // Usuarios agregando un registro nuevo: user -> color
            this.usersAdding = new Map();

            // Mi propio lock activo (lo que yo estoy editando en este momento).
            this.myEditingRowId = null;

            // Caché de colores por usuario.
            this.userColors = new Map();

            // Timer de watchdog que vigila que los indicadores no desaparezcan.
            this.watchdogTimer = null;
        }

        // ─── Ciclo de vida ────────────────────────────────────────

        start() {
            this._connect();
            this._startWatchdog();
            this._bindLifecycleEvents();
        }

        stop() {
            if (this.watchdogTimer) clearInterval(this.watchdogTimer);
            if (this.socket) try { this.socket.close(); } catch (_) {}
        }

        // ─── Conexión WebSocket ───────────────────────────────────

        _buildUrl() {
            const proto = location.protocol === 'https:' ? 'wss:' : 'ws:';
            const isLocal = location.hostname === 'localhost' || location.hostname === '127.0.0.1';
            // Detectar automáticamente el virtual path usando la base del documento
            // (funciona tanto en localhost sin subpath como en /CartaPorte/ en producción).
            const path = location.pathname.replace(/\/[^\/]*$/, '');
            const segments = path.split('/').filter(Boolean);
            const basePath = isLocal ? '' : (segments.length > 0 ? '/' + segments[0] : '');
            const channel = this.crud.config.wsChannel || this.crud.config.tableId || 'crud';
            return `${proto}//${location.host}${basePath}/WebSocket/Connect/${channel}`;
        }

        _connect() {
            if (this.socket) return;
            const url = this._buildUrl();

            // Suprimir el error "WebSocket connection failed" en consola cuando
            // estamos reintentando: es esperado y no bloquea la aplicación.
            try {
                this.socket = new WebSocket(url);
                this.socket.onopen = () => {
                    this.reconnectAttempts = 0;
                    this._updateStatusIndicator(true);
                };
                this.socket.onclose = (e) => {
                    this.socket = null;
                    this._updateStatusIndicator(false);
                    // Si el servidor rechazó definitivamente (404, 501), no reintentar infinitamente.
                    if (e && (e.code === 1002 || e.code === 1003 || e.code === 1008)) {
                        this.reconnectAttempts = RECONNECT_MAX_ATTEMPTS;
                        return;
                    }
                    this._scheduleReconnect();
                };
                this.socket.onerror = () => {
                    // Silenciar: el onclose siguiente se encarga de reconectar.
                    this._updateStatusIndicator(false);
                };
                this.socket.onmessage = (e) => this._handleIncomingMessage(e);
            } catch (ex) {
                this._updateStatusIndicator(false);
                this._scheduleReconnect();
            }
        }

        _scheduleReconnect() {
            if (this.reconnectAttempts >= RECONNECT_MAX_ATTEMPTS) return;
            const delay = Math.min(RECONNECT_BASE_MS * Math.pow(1.5, this.reconnectAttempts), RECONNECT_MAX_MS);
            this.reconnectAttempts++;
            setTimeout(() => this._connect(), delay);
        }

        _send(payload) {
            if (!this.socket || this.socket.readyState !== WebSocket.OPEN) return;
            payload.table = this.crud.config.wsChannel || this.crud.config.tableId;
            payload.user = this.me;
            this.socket.send(JSON.stringify(payload));
        }

        // ─── Acciones públicas (llamadas desde CrudTable) ─────────

        notifyEditing(rowId) {
            this.myEditingRowId = String(rowId);
            this._send({ action: 'lock', id: rowId });
        }

        notifyDoneEditing(rowId) {
            this.myEditingRowId = null;
            this._send({ action: 'unlock', id: rowId });
        }

        notifyAdding() {
            this._send({ action: 'adding', id: '__new__' });
        }

        notifyDoneAdding() {
            this._send({ action: 'stopadding', id: '__new__' });
        }

        notifyDataChanged() {
            this._send({ action: 'refresh' });
        }

        isRowLockedByOther(rowId) {
            return this.rowsBeingEdited.has(String(rowId));
        }

        // ─── Recepción de mensajes ────────────────────────────────

        _handleIncomingMessage(e) {
            let msg;
            try { msg = JSON.parse(e.data); } catch (_) { return; }

            // Ignorar mensajes del protocolo de transferencia de archivos.
            if (msg.tipo && msg.size !== undefined) return;

            const myTable = this.crud.config.wsChannel || this.crud.config.tableId;
            if (msg.table && msg.table !== myTable) return;

            const senderIsMe = (msg.user || '').toLowerCase() === this.me.toLowerCase();

            // Entregar una copia al hook global, si la vista se suscribio.
            // Esto permite que componentes fuera del CrudTable (por ejemplo,
            // el countdown sincronizado de Facturacion) reaccionen a eventos
            // custom sin modificar este archivo.
            if (typeof window !== 'undefined' && typeof window.onCrudWsMessage === 'function') {
                try { window.onCrudWsMessage(msg, { senderIsMe, channel: myTable }); } catch (_) {}
            }

            switch (msg.action) {
                case 'lock':
                    if (!senderIsMe) this._showRowAsLocked(msg.id, msg.user);
                    break;
                case 'unlock':
                    this._showRowAsUnlocked(msg.id);
                    break;
                case 'adding':
                    if (!senderIsMe) this._showUserAsAdding(msg.user);
                    break;
                case 'stopadding':
                    this._showUserAsNotAdding(msg.user);
                    break;
                case 'refresh':
                    if (!senderIsMe) setTimeout(() => this._requestSafeRefresh(), 50);
                    break;
                // Eventos custom (Facturacion / Embarques). Todos provocan
                // un refresh de la tabla en los demas clientes — asi la UI
                // refleja el cambio sin esperar al polling.
                case 'Subir partes':
                case 'Registrar clase':
                    if (!senderIsMe) setTimeout(() => this._requestSafeRefresh(), 50);
                    break;
            }
        }

        _requestSafeRefresh() {
            if (!this.crud) return;
            if (typeof this.crud.hasActiveEditSession === 'function' && this.crud.hasActiveEditSession()) {
                if (typeof this.crud.queueRemoteRefresh === 'function') this.crud.queueRemoteRefresh();
                return;
            }

            this.crud._refresh();
        }

        // ─── Estado visual: filas bloqueadas ──────────────────────

        _showRowAsLocked(rowId, user) {
            const id = String(rowId);
            const color = this._getColorForUser(user);
            this.rowsBeingEdited.set(id, { user, color });
            this._paintLockedRow(id);
            this._refreshActiveUsersPanel();
        }

        _showRowAsUnlocked(rowId) {
            const id = String(rowId);
            this.rowsBeingEdited.delete(id);
            this._unpaintLockedRow(id);
            this._refreshActiveUsersPanel();
        }

        _paintLockedRow(rowId) {
            const tr = this._findRow(rowId);
            if (!tr) return;

            const info = this.rowsBeingEdited.get(rowId);
            if (!info) return;

            // Limpiar indicador previo si existiera.
            const oldIndicator = tr.querySelector('.ct-lock-indicator');
            if (oldIndicator) oldIndicator.remove();

            tr.dataset.locked = 'true';
            tr.dataset.lockedBy = info.user;
            tr.classList.add('ct-row-locked');
            tr.style.setProperty('--lock-color', info.color);
            tr.title = `🔒 ${info.user} está editando este registro`;

            const indicator = document.createElement('div');
            indicator.className = 'ct-lock-indicator';
            indicator.innerHTML =
                `<div class="ct-lock-dot" style="background:${info.color};--lock-color:${info.color}"></div>` +
                `<span class="ct-lock-user" style="background:${info.color}">${this._escapeHtml(info.user)}</span>`;

            const firstCell = tr.querySelector('td');
            if (firstCell) {
                firstCell.style.position = 'relative';
                firstCell.insertBefore(indicator, firstCell.firstChild);
            }
        }

        _unpaintLockedRow(rowId) {
            const tr = this._findRow(rowId);
            if (!tr) return;
            tr.dataset.locked = 'false';
            delete tr.dataset.lockedBy;
            tr.classList.remove('ct-row-locked');
            tr.style.removeProperty('--lock-color');
            tr.title = '';
            const indicator = tr.querySelector('.ct-lock-indicator');
            if (indicator) indicator.remove();
        }

        // Re-pinta todos los locks. Llamar después de cualquier re-render de la tabla.
        repaintAllLocks() {
            if (this.rowsBeingEdited.size === 0) return;
            for (const [id] of this.rowsBeingEdited) {
                this._paintLockedRow(id);
            }
        }

        // ─── Estado visual: panel de usuarios activos ─────────────

        _showUserAsAdding(user) {
            this.usersAdding.set(user, this._getColorForUser(user));
            this._refreshActiveUsersPanel();
        }

        _showUserAsNotAdding(user) {
            this.usersAdding.delete(user);
            this._refreshActiveUsersPanel();
        }

        _refreshActiveUsersPanel() {
            const panelId = this._panelId();
            let panel = document.getElementById(panelId);

            const nothingToShow = this.rowsBeingEdited.size === 0 && this.usersAdding.size === 0;
            if (nothingToShow) {
                if (panel) panel.remove();
                return;
            }

            if (!panel) {
                const container = this._tableContainer();
                if (!container) return;
                panel = document.createElement('div');
                panel.id = panelId;
                panel.className = 'ct-active-users-panel';
                container.insertBefore(panel, container.firstChild);
            }

            // Construir resumen por usuario.
            const summary = new Map();
            for (const [, info] of this.rowsBeingEdited) {
                if (!summary.has(info.user)) summary.set(info.user, { color: info.color, editingCount: 0, isAdding: false });
                summary.get(info.user).editingCount++;
            }
            for (const [user, color] of this.usersAdding) {
                if (!summary.has(user)) summary.set(user, { color, editingCount: 0, isAdding: false });
                summary.get(user).isAdding = true;
            }

            let html = '<div class="ct-aup-title">' +
                '<svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">' +
                '<path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2"/><circle cx="9" cy="7" r="4"/>' +
                '<path d="M23 21v-2a4 4 0 0 0-3-3.87"/><path d="M16 3.13a4 4 0 0 1 0 7.75"/>' +
                '</svg> En uso:</div>';

            for (const [user, data] of summary) {
                let activityLabel = '';
                if (data.editingCount > 0 && data.isAdding) {
                    activityLabel = `editando ${data.editingCount} + agregando nuevo`;
                } else if (data.editingCount > 0) {
                    activityLabel = data.editingCount === 1 ? 'editando' : `editando (${data.editingCount})`;
                } else if (data.isAdding) {
                    activityLabel = 'agregando nuevo';
                }
                html += `<div class="ct-aup-user">` +
                    `<span class="ct-aup-dot" style="background:${data.color}"></span>` +
                    `${this._escapeHtml(user)} <span class="ct-aup-count">${activityLabel}</span></div>`;
            }

            panel.innerHTML = html;
        }

        // ─── Indicador de conexión (punto verde/rojo en el header) ─

        _updateStatusIndicator(isConnected) {
            const indicatorId = this._statusId();
            let indicator = document.getElementById(indicatorId);

            if (!indicator) {
                const table = document.getElementById(this.crud.config.tableId);
                const header = table && table.closest('.card') && table.closest('.card').querySelector('.card-header');
                if (!header) return;
                indicator = document.createElement('div');
                indicator.id = indicatorId;
                indicator.className = 'ct-ws-indicator';
                indicator.innerHTML = '<span class="ct-ws-dot"></span><span class="ct-ws-label"></span>';
                header.style.position = 'relative';
                header.appendChild(indicator);
            }

            const dot = indicator.querySelector('.ct-ws-dot');
            const label = indicator.querySelector('.ct-ws-label');
            dot.classList.toggle('ct-ws-connected', isConnected);
            dot.classList.toggle('ct-ws-disconnected', !isConnected);
            label.textContent = isConnected ? 'En vivo' : 'Desconectado';
        }

        // ─── Vigilancia y reconexión en cambio de pestaña ─────────

        _startWatchdog() {
            if (this.watchdogTimer) return;
            // Cada 1.5s revisa si los indicadores siguen visibles.
            // Si alguno falta (por redibujado de DataTables), lo repinta.
            this.watchdogTimer = setInterval(() => {
                if (document.hidden) return;
                if (this.rowsBeingEdited.size === 0) return;
                const table = document.getElementById(this.crud.config.tableId);
                if (!table) return;

                for (const [id] of this.rowsBeingEdited) {
                    const tr = table.querySelector(`tr[data-id="${id}"]`);
                    if (tr && !tr.querySelector('.ct-lock-indicator')) {
                        this._paintLockedRow(id);
                    }
                }
            }, WATCHDOG_INTERVAL_MS);
        }

        _bindLifecycleEvents() {
            // Cuando el usuario vuelve a la pestaña, repintamos todo y reconectamos si hace falta.
            document.addEventListener('visibilitychange', () => {
                if (document.hidden) return;
                this.repaintAllLocks();
                this._refreshActiveUsersPanel();
                if (!this.socket || this.socket.readyState !== WebSocket.OPEN) {
                    this.reconnectAttempts = 0;
                    this._connect();
                }
            });

            // Si el usuario cierra la ventana mientras edita, liberar su lock.
            window.addEventListener('beforeunload', () => {
                if (this.myEditingRowId) this.notifyDoneEditing(this.myEditingRowId);
            });
        }

        // ─── Utilidades internas ──────────────────────────────────

        _getColorForUser(user) {
            const key = (user || '').toLowerCase();
            if (this.userColors.has(key)) return this.userColors.get(key);

            // Asignación por hash determinista del nombre: GARANTIZA que el mismo usuario
            // tenga el mismo color en todos los navegadores, y que nombres distintos
            // tengan alta probabilidad de obtener colores diferentes.
            // Se usa un mejor algoritmo de hash (djb2) para mejor distribución.
            let hash = 5381;
            for (let i = 0; i < key.length; i++) {
                hash = ((hash << 5) + hash) + key.charCodeAt(i); // hash * 33 + c
                hash = hash & 0xffffffff;
            }

            // Convertir a índice positivo en la paleta
            let idx = Math.abs(hash) % USER_COLOR_PALETTE.length;

            // Si ese color ya está en uso por OTRO usuario, buscar el siguiente disponible
            const usedByOthers = new Set();
            for (const [otherKey, otherColor] of this.userColors) {
                if (otherKey !== key) usedByOthers.add(otherColor);
            }

            let chosen = USER_COLOR_PALETTE[idx];
            let attempts = 0;
            while (usedByOthers.has(chosen) && attempts < USER_COLOR_PALETTE.length) {
                idx = (idx + 1) % USER_COLOR_PALETTE.length;
                chosen = USER_COLOR_PALETTE[idx];
                attempts++;
            }

            this.userColors.set(key, chosen);
            return chosen;
        }

        _findRow(rowId) {
            const table = document.getElementById(this.crud.config.tableId);
            return table ? table.querySelector(`tr[data-id="${rowId}"]`) : null;
        }

        _tableContainer() {
            const table = document.getElementById(this.crud.config.tableId);
            return table ? table.closest('.card-body') : null;
        }

        _panelId() {
            return this.crud._scopeId('ct-active-users');
        }

        _statusId() {
            return this.crud._scopeId('ct-ws-status');
        }

        _escapeHtml(s) {
            if (s == null) return '';
            return String(s)
                .replace(/&/g, '&amp;').replace(/</g, '&lt;')
                .replace(/>/g, '&gt;').replace(/"/g, '&quot;');
        }
    }

    window.CrudTableRealTime = CrudTableRealTime;
})();
