# Tables-para-estudiar
//////////////////// CrudTable.js — Motor genérico de tablas CRUD con edición inline. ////////////////////
//
//  Para crear una nueva vista de tabla solo se necesita:
//    1. Un HTML con la estructura base (card + table#tableData).
//    2. Un objeto tableConfig con urls, columns, y opcionalmente filters/hooks.
//    3. Llamar:  new CrudTable(tableConfig).init();
//
//  Opciones de configuración:
//    tableId  — id del <table> en el HTML (por defecto 'tableData').
//    urls     — { getAll, insert, update, delete, [availableEntries] }
//    columns  — array de definiciones de columna.
//    filters  — { serverSide, dateKey, statusKey, selectFilter } — barra de filtros (opcional).
//    hooks    — { beforeInsert(payload): Promise<bool> } — validaciones extra (opcional).
////////////////////

class CrudTable {

    //////////////////// Constructor ////////////////////

    constructor(config) {
        this.config       = config;
        this.tableData    = [];
        this.selectedRow  = null;
        this.dataTableInstance = null;
        this.creatingNewRow    = false;
        this._rowDataMap       = new Map();
        this._savedPage        = 0;

        this.filterFrom        = null;
        this.filterTo          = null;
        this.showInactiveOnly  = false;
        this.showActiveOnly    = false;
        this.filterSelectValue = '';
        this._filtersRendered  = false;
        this.pendingRemoteRefresh = false;

        // Módulo opcional de colaboración en tiempo real. Se activa en init().
        this.realTime = null;
    }

    /** Genera un ID con alcance único por tabla para evitar colisiones en vistas con múltiples instancias. */
    _scopeId(name) {
        return `${this.config.tableId}_${name}`;
    }

    //////////////////// Ciclo de vida ////////////////////

    /**
     * Inicializa (o recarga) la tabla.
     * @param {string} [fromDate] — fecha inicio (solo para filtrado del lado del servidor).
     * @param {string} [toDate]   — fecha fin.
     */
    async init(fromDate, toDate) {
        await this._fetchDynamicOptions();
        await this._fetchFilterOptions();

        /* Si se pasan fechas explícitas, actualizar el estado del filtro */
        if (fromDate && toDate) {
            this.filterFrom = fromDate;
            this.filterTo   = toDate;
        }

        /* Renderizar la barra de filtros (solo la primera vez) */
        this._initFilters();

        /* Obtener datos del servidor */
        if (this.config.filters && this.config.filters.serverSide) {
            this.tableData = await this._fetchData(this.filterFrom, this.filterTo);
        } else {
            this.tableData = await this._fetchData();
        }

        this._renderTable();
        this._attachButtons();
        this._initShortcuts();

        // Activar colaboración en tiempo real si el módulo está disponible.
        if (!this.realTime && typeof CrudTableRealTime !== 'undefined') {
            this.realTime = new CrudTableRealTime(this);
            this.realTime.start();
        }

        /* Si se guardó con Ctrl+Enter, abrir nueva fila automáticamente */
        if (this._addRowAfterSave) {
            this._addRowAfterSave = false;
            this.addRow();
        }
    }

    hasActiveEditSession() {
        const table = document.getElementById(this.config.tableId);
        if (!table) return this.creatingNewRow;
        return this.creatingNewRow || !!table.querySelector('tr.row-editing');
    }

    queueRemoteRefresh() {
        this.pendingRemoteRefresh = true;
    }

    async flushRemoteRefresh() {
        if (!this.pendingRemoteRefresh) return;
        if (this.hasActiveEditSession()) return;
        this.pendingRemoteRefresh = false;
        await this._refresh();
    }

    /**
     * Recarga ligera: solo obtiene datos del servidor y redibuja la tabla.
     * Omite _fetchDynamicOptions, _fetchFilterOptions, _initFilters (barra), _attachButtons, _initShortcuts
     * que ya se ejecutaron en el init() inicial.  Usar después de guardar/eliminar.
     */
    async _refresh() {
        await this._fetchDynamicOptions();  // recargar opciones dinámicas (precintos, cajas, etc.)
        this._initFilters();   // leer el estado actual de los filtros del DOM
        if (this.config.filters && this.config.filters.serverSide) {
            this.tableData = await this._fetchData(this.filterFrom, this.filterTo);
        } else {
            this.tableData = await this._fetchData();
        }
        this._renderTable();

        // Repintar locks tras recargar datos.
        if (this.realTime) this.realTime.repaintAllLocks();

        if (this._addRowAfterSave) {
            this._addRowAfterSave = false;
            this.addRow();
        }
    }


    //////////////////// SECCIÓN 1 — Comunicación con el servidor ////////////////////
    // Peticiones GET (lectura) y POST (escritura).

    /** Trae todos los registros. Opcionalmente filtra por rango de fechas en el servidor. */
    async _fetchData(fromDate, toDate) {
        const url = new URL(this.config.urls.getAll, window.location.origin);
        if (fromDate) url.searchParams.append('fromDate', fromDate);
        if (toDate)   url.searchParams.append('toDate',   toDate);

        // Filtro de texto con paramName configurable (ej: "loadFilter").
        if (this.filterTextValue && this.config.filters && this.config.filters.textFilter) {
            const pName = this.config.filters.textFilter.paramName || 'filter';
            url.searchParams.append(pName, this.filterTextValue);
        }

        const res = await fetch(url.toString());
        return await res.json();
    }

    /** Carga las opciones de los <select> que tienen dataUrl (catálogos dinámicos). */
    async _fetchDynamicOptions() {
        const dynamicCols = this.config.columns.filter(c => c.input && c.input.dataUrl);
        if (!dynamicCols.length) return;
        await Promise.all(dynamicCols.map(async (col) => {
            const res  = await fetch(col.input.dataUrl);
            const data = await res.json();
            col.input.options = data.map(x => ({
                value: x[col.input.valueField || 'Id'],
                label: x[col.input.labelField || 'Descripcion']
            }));
        }));
    }

    /** Envía datos al servidor vía POST (insertar, actualizar, eliminar). */
    async _postRequest(url, body) {
        const res = await fetch(url, {
            method: 'POST',
            headers: { 'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8' },
            body
        });
        if (!res.ok) return { ok: false, message: 'Error en la petición.' };

        const json = await res.json().catch(() => null);
        if (json && json.success === false)
            return { ok: false, message: json.message || 'Error en el servidor.' };

        return { ok: true };
    }


    //////////////////// SECCIÓN 1.5 — Filtros de fecha y status ////////////////////
    // Barra visual para acotar registros por rango de fechas
    // y un botón para ver solo los inactivos.

    /** Prepara los filtros: calcula la semana actual y pinta la barra (una sola vez). */
    _initFilters() {
        if (!this.config.filters) return;

        if (!this._filtersRendered) {
            if (this.config.filters.dateKey && (!this.filterFrom || !this.filterTo)) this._setDefaultDateRange();
            this._renderFilterBar();
            this._filtersRendered = true;
        } else {
            const fromInput = document.getElementById(this._scopeId('filterFrom'));
            const toInput   = document.getElementById(this._scopeId('filterTo'));
            const btn       = document.getElementById(this._scopeId('btnInactivos'));
            const btnAct    = document.getElementById(this._scopeId('btnActivos'));
            const sel       = document.getElementById(this._scopeId('filterSelect'));
            const txt       = document.getElementById(this._scopeId('filterText'));
            if (fromInput) this.filterFrom        = fromInput.value;
            if (toInput)   this.filterTo          = toInput.value;
            if (btn)       this.showInactiveOnly  = btn.classList.contains('active');
            if (btnAct)    this.showActiveOnly    = btnAct.classList.contains('active');
            if (sel)       this.filterSelectValue = sel.value;
            if (txt)       this.filterTextValue   = txt.value;
        }
    }

    /** Calcula lunes–domingo de la semana actual. */
    _setDefaultDateRange() {
        const today = new Date();
        const day   = today.getDay();
        const diff  = (day === 0 ? -6 : 1) - day;
        const monday = new Date(today);
        monday.setDate(today.getDate() + diff);
        const sunday = new Date(monday);
        sunday.setDate(monday.getDate() + 6);
        this.filterFrom = monday.toISOString().slice(0, 10);
        this.filterTo   = sunday.toISOString().slice(0, 10);
    }

    /** Carga las opciones del filtro tipo select desde el servidor. */
    async _fetchFilterOptions() {
        const sf = this.config.filters && this.config.filters.selectFilter;
        if (!sf || !sf.dataUrl || sf._optionsFetched) return;
        const res = await fetch(sf.dataUrl);
        const data = await res.json();
        sf.options = data.map(x => ({
            value: x[sf.valueField || 'Id'],
            label: x[sf.labelField || 'Nombre']
        }));
        sf._optionsFetched = true;
    }

    /** Inyecta el HTML de la barra de filtros antes de la tabla. */
    _renderFilterBar() {
        const table     = document.getElementById(this.config.tableId);
        const container = table.parentElement;
        const filters   = this.config.filters;

        const bar = document.createElement('div');
        bar.id = this._scopeId('filterBar');
        bar.className = 'filterBar';

        let bodyHTML = '';

        if (filters.selectFilter) {
            bodyHTML += `
                <div class="filter-group">
                    <label for="${this._scopeId('filterSelect')}">${filters.selectFilter.label}</label>
                    <select id="${this._scopeId('filterSelect')}" class="filter-select">
                        <option value="">Todos</option>
                    </select>
                </div>`;
        }

        if (filters.dateKey) {
            bodyHTML += `
                <div class="filter-group">
                    <label for="${this._scopeId('filterFrom')}">Fecha Desde</label>
                    <input type="date" id="${this._scopeId('filterFrom')}" value="${this.filterFrom || ''}">
                </div>
                <div class="filter-group">
                    <label for="${this._scopeId('filterTo')}">Fecha Hasta</label>
                    <input type="date" id="${this._scopeId('filterTo')}" value="${this.filterTo || ''}">
                </div>
                <div class="filter-group" style="align-self:flex-end">
                    <button id="${this._scopeId('btnFilter')}" class="btn-filter-apply">
                        <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24"
                             fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
                            <circle cx="11" cy="11" r="8"></circle><line x1="21" y1="21" x2="16.65" y2="16.65"></line>
                        </svg>
                        Filtrar
                    </button>
                </div>`;
        }

        if (filters.statusKey) {
            if (filters.dateKey || filters.selectFilter) {
                bodyHTML += `<div class="filter-divider"></div>`;
            }
            bodyHTML += `
                <div class="filter-group" style="align-self:flex-end">
                    <button id="${this._scopeId('btnActivos')}" class="btn-filter-inactive">
                        <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24"
                             fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
                            <path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"></path>
                            <polyline points="22 4 12 14.01 9 11.01"></polyline>
                        </svg>
                        Mostrar solo Activos
                    </button>
                </div>
                <div class="filter-group" style="align-self:flex-end">
                    <button id="${this._scopeId('btnInactivos')}" class="btn-filter-inactive">
                        <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24"
                             fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
                            <circle cx="12" cy="12" r="10"></circle><line x1="15" y1="9" x2="9" y2="15"></line>
                            <line x1="9" y1="9" x2="15" y2="15"></line>
                        </svg>
                        Mostrar solo Inactivos
                    </button>
                </div>`;
        }

        // Filtro de texto con autocomplete opcional (datalist HTML nativo).
        if (filters.textFilter) {
            const tfDlId = this._scopeId('filterTextDL');
            const tfListAttr = filters.textFilter.autocompleteUrl ? `list="${tfDlId}"` : '';
            bodyHTML += `
                <div class="filter-group">
                    <label for="${this._scopeId('filterText')}">${filters.textFilter.label || 'Buscar'}</label>
                    <input type="text" id="${this._scopeId('filterText')}" class="filter-select"
                           placeholder="${filters.textFilter.placeholder || ''}"
                           value="${this.filterTextValue || ''}"
                           style="min-width:180px;" ${tfListAttr} autocomplete="off">
                    ${filters.textFilter.autocompleteUrl ? `<datalist id="${tfDlId}"></datalist>` : ''}
                </div>
                <div class="filter-group" style="align-self:flex-end">
                    <button id="${this._scopeId('btnTextFilter')}" class="btn-filter-apply">
                        <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24"
                             fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
                            <circle cx="11" cy="11" r="8"></circle><line x1="21" y1="21" x2="16.65" y2="16.65"></line>
                        </svg>
                        Filtrar
                    </button>
                </div>`;
        }

        // Botones personalizados (ej: "Subir numeros de parte" en Facturacion).
        // Toggle de columnas opcionales (switches deslizables)
        if (Array.isArray(filters.columnToggles) && filters.columnToggles.length > 0) {
            const togglesId = this._scopeId('columnToggles');
            let togglesHtml = `
                <div class="filter-group ct-col-toggles">
                    <label style="font-size:.72rem;color:#666;font-weight:600;margin-bottom:4px;display:block;">Columnas adicionales</label>
                    <div id="${togglesId}" class="ct-col-toggle-switches">
                        <label class="ct-switch ct-switch-all">
                            <span class="ct-switch-label">Todas</span>
                            <input type="checkbox" id="${this._scopeId('colToggleAll')}">
                            <span class="ct-switch-track"><span class="ct-switch-thumb"></span></span>
                        </label>`;
            filters.columnToggles.forEach(t => {
                const col = this.config.columns.find(c => c.key === t.key);
                const checked = col && col.visible ? 'checked' : '';
                togglesHtml += `
                    <label class="ct-switch">
                        <span class="ct-switch-label">${t.label}</span>
                        <input type="checkbox" data-col-key="${t.key}" ${checked}>
                        <span class="ct-switch-track"><span class="ct-switch-thumb"></span></span>
                    </label>`;
            });
            togglesHtml += `</div></div>`;
            bodyHTML += togglesHtml;
        }

        // Botones personalizados (ej: "Subir numeros de parte") — despues de los toggles
        if (Array.isArray(filters.customButtons) && filters.customButtons.length > 0) {
            filters.customButtons.forEach(btn => {
                bodyHTML += `
                    <div class="filter-group" style="align-self:flex-end">
                        <button id="${this._scopeId('customBtn_' + btn.id)}"
                                class="${btn.className || 'btn-filter-apply'}">
                            ${btn.label}
                        </button>
                    </div>`;
            });
        }

        const singleRowClass = (filters.textFilter && Array.isArray(filters.columnToggles) && filters.columnToggles.length > 0)
            ? ' ct-single-row' : '';

        bar.innerHTML = `
            <div class="filter-title">
                <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 24 24"
                     fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round">
                    <polygon points="22 3 2 3 10 12.46 10 19 14 21 14 12.46 22 3"></polygon>
                </svg>
                Filtros
            </div>
            <div class="filter-body${singleRowClass}">${bodyHTML}</div>`;
        container.insertBefore(bar, table);

        if (filters.selectFilter) {
            const sel = document.getElementById(this._scopeId('filterSelect'));
            (filters.selectFilter.options || []).forEach(opt => {
                const o = document.createElement('option');
                o.value = opt.value; o.textContent = opt.label;
                sel.appendChild(o);
            });
            if (this.filterSelectValue) sel.value = this.filterSelectValue;
            sel.addEventListener('change', () => {
                this.filterSelectValue = sel.value;
                this._renderTable();
            });
        }

        if (filters.dateKey) {
            document.getElementById(this._scopeId('btnFilter')).addEventListener('click', async () => {
                const from = document.getElementById(this._scopeId('filterFrom')).value;
                const to   = document.getElementById(this._scopeId('filterTo')).value;
                if (!from || !to) { alert('Seleccione ambas fechas.'); return; }
                if (from > to)    { alert('La fecha "Desde" no puede ser mayor a "Hasta".'); return; }
                this.showInactiveOnly = false;
                this.showActiveOnly = false;
                const btn = document.getElementById(this._scopeId('btnInactivos'));
                if (btn) btn.classList.remove('active');
                const btnAct = document.getElementById(this._scopeId('btnActivos'));
                if (btnAct) btnAct.classList.remove('active');

                if (filters.serverSide) {
                    await this.init(from, to);
                } else {
                    this.filterFrom = from;
                    this.filterTo   = to;
                    this._renderTable();
                }
            });
        }

        if (filters.statusKey) {
            document.getElementById(this._scopeId('btnActivos')).addEventListener('click', () => {
                this.showActiveOnly = !this.showActiveOnly;
                if (this.showActiveOnly) this.showInactiveOnly = false;
                const btnAct = document.getElementById(this._scopeId('btnActivos'));
                const btnInact = document.getElementById(this._scopeId('btnInactivos'));
                btnAct.classList.toggle('active', this.showActiveOnly);
                btnInact.classList.remove('active');
                this._renderTable();
            });

            document.getElementById(this._scopeId('btnInactivos')).addEventListener('click', () => {
                this.showInactiveOnly = !this.showInactiveOnly;
                if (this.showInactiveOnly) this.showActiveOnly = false;
                const btnInact = document.getElementById(this._scopeId('btnInactivos'));
                const btnAct = document.getElementById(this._scopeId('btnActivos'));
                btnInact.classList.toggle('active', this.showInactiveOnly);
                btnAct.classList.remove('active');
                this._renderTable();
            });
        }

        // Handler del filtro de texto + carga del datalist de autocomplete.
        if (filters.textFilter) {
            const txtInput = document.getElementById(this._scopeId('filterText'));
            const btnTxtFilter = document.getElementById(this._scopeId('btnTextFilter'));
            const doTextFilter = async () => {
                this.filterTextValue = txtInput.value.trim();
                if (filters.serverSide) {
                    await this._refresh();
                } else {
                    this._renderTable();
                }
            };
            if (txtInput) {
                txtInput.addEventListener('keydown', (e) => { if (e.key === 'Enter') doTextFilter(); });
                txtInput.addEventListener('change', doTextFilter);
            }
            if (btnTxtFilter) btnTxtFilter.addEventListener('click', doTextFilter);

            // Cargar opciones de autocomplete si están configuradas.
            if (filters.textFilter.autocompleteUrl) {
                const dl = document.getElementById(this._scopeId('filterTextDL'));
                if (dl) {
                    fetch(filters.textFilter.autocompleteUrl)
                        .then(r => r.json())
                        .then(data => {
                            dl.innerHTML = '';
                            (data || []).forEach(item => {
                                const opt = document.createElement('option');
                                opt.value = item.Nombre || item.Id || item;
                                dl.appendChild(opt);
                            });
                        })
                        .catch(() => {});
                }
            }
        }

        // Handlers de botones personalizados.
        if (Array.isArray(filters.customButtons)) {
            filters.customButtons.forEach(btn => {
                const el = document.getElementById(this._scopeId('customBtn_' + btn.id));
                if (el && typeof btn.onClick === 'function') {
                    el.addEventListener('click', () => btn.onClick(this));
                }
            });
        }

        // Handlers de toggles de columnas.
        if (Array.isArray(filters.columnToggles) && filters.columnToggles.length > 0) {
            const container = document.getElementById(this._scopeId('columnToggles'));
            const cbAll = document.getElementById(this._scopeId('colToggleAll'));
            if (container) {
                const cbs = Array.from(container.querySelectorAll('input[type="checkbox"][data-col-key]'));

                const toggleColumn = (key, visible) => {
                    const col = this.config.columns.find(c => c.key === key);
                    if (col) col.visible = visible;
                };

                cbs.forEach(cb => {
                    cb.addEventListener('change', () => {
                        toggleColumn(cb.dataset.colKey, cb.checked);
                        if (cbAll) cbAll.checked = cbs.every(x => x.checked);
                        this._renderTable();
                    });
                });

                if (cbAll) {
                    // Estado inicial del "Seleccionar todas"
                    cbAll.checked = cbs.length > 0 && cbs.every(x => x.checked);
                    cbAll.addEventListener('change', () => {
                        cbs.forEach(cb => {
                            cb.checked = cbAll.checked;
                            toggleColumn(cb.dataset.colKey, cb.checked);
                        });
                        this._renderTable();
                    });
                }
            }
        }
    }

    /**
     * Devuelve los datos filtrados según el estado actual de la barra.
     * - Si no hay filtros configurados, devuelve todos los datos.
     * - selectFilter filtra por el valor del dropdown.
     * - "Mostrar solo Inactivos" siempre se aplica del lado del cliente.
     * - Si serverSide=true, las fechas ya vienen filtradas del servidor.
     * - Si serverSide=false, se filtran aquí en el navegador.
     */
    _getFilteredData() {
        if (!this.config.filters) return this.tableData;
        const { dateKey, statusKey, selectFilter } = this.config.filters;

        let data = this.tableData;

        if (selectFilter && this.filterSelectValue) {
            data = data.filter(row => String(row[selectFilter.key]) === String(this.filterSelectValue));
        }

        if (this.showActiveOnly && statusKey) {
            return data.filter(row => !!row[statusKey]);
        }

        if (this.showInactiveOnly && statusKey) {
            return data.filter(row => !row[statusKey]);
        }

        if (this.config.filters.serverSide) return data;

        if (!dateKey) return data;
        return data.filter(row => {
            const dateStr = this._parseMsDate(row[dateKey]);
            if (!dateStr) return false;
            if (this.filterFrom && dateStr < this.filterFrom) return false;
            if (this.filterTo   && dateStr > this.filterTo)   return false;
            return true;
        });
    }


    //////////////////// SECCION 2 — Serialización de datos ////////////////////
    // Convierte un registro JS → payload → formulario URL-encoded.

    /** Construye un objeto plano con las columnas que deben persistirse. */
    _buildPayload(row) {
        const payload = {};
        this.config.columns.forEach(col => {
            if (!col.persist) return;
            const type = col.input && col.input.type;
            payload[col.key] = (type === 'date') ? this._parseMsDate(row[col.key]) : row[col.key];
            if (col.allowNewEntry && col.displayKey && row[col.displayKey] !== undefined) {
                payload[col.displayKey] = row[col.displayKey];
            }
        });
        return payload;
    }

    /** Convierte un objeto plano a URLSearchParams para enviar por POST. */
    _payloadToForm(payload) {
        const params = new URLSearchParams();
        Object.keys(payload).forEach(k => {
            const v = payload[k];
            if (v === undefined || v === null)  params.append(k, '');
            else if (typeof v === 'boolean')    params.append(k, v ? 'true' : 'false');
            else                                params.append(k, v.toString());
        });
        return params;
    }


    //////////////////// SECCIÓN 3 — Renderizado de la tabla ////////////////////
    // Dibuja <thead>, <tbody> y activa DataTables.

    /** Limpia y redibuja toda la tabla con los datos filtrados. */
    _renderTable() {
        const table = document.getElementById(this.config.tableId);
        table.classList.add('crud-table');

        /* Destruir DataTable ANTES de limpiar el DOM */
        this._destroyDataTable();

        table.innerHTML = '';

        /* Encabezados */
        const thead   = document.createElement('thead');
        const headRow = document.createElement('tr');
        // Columnas que aparecen como secundarias dentro de un groupWith: ocultarlas del header/body
        const groupedSecondary = new Set();
        this.config.columns.forEach(col => {
            if (col.groupWith) groupedSecondary.add(col.groupWith);
        });
        this.config.columns.forEach(col => {
            if (col.visible === false) return;
            if (groupedSecondary.has(col.key)) return; // la mostramos dentro de su columna principal
            const th = document.createElement('th');
            th.textContent = col.label;
            headRow.appendChild(th);
        });
        thead.appendChild(headRow);
        table.appendChild(thead);

        /* Cuerpo con filas filtradas — guardar referencia en mapa para delegación */
        const tbody = document.createElement('tbody');
        let filteredData = this._getFilteredData();
        // removeIf: eliminar registros que ya no deben mostrarse (ej: cambiaron de status)
        if (typeof this.config.removeIf === 'function') {
            filteredData = filteredData.filter(row => !this.config.removeIf(row));
        }
        this._rowDataMap = new Map();
        const fragment = document.createDocumentFragment();
        filteredData.forEach(row => {
            this._rowDataMap.set(String(row.Id), row);
            fragment.appendChild(this._renderRow(row));
        });
        tbody.appendChild(fragment);
        table.appendChild(tbody);

        /* Eventos delegados: sobreviven al filtrado/ordenamiento interno de DataTables */
        tbody.addEventListener('click', (e) => {
            const tr = e.target.closest('tr');
            if (tr && !tr.classList.contains('table-warning')) this._selectRow(tr);
        });
        tbody.addEventListener('dblclick', (e) => {
            const tr = e.target.closest('tr');
            if (!tr || tr.classList.contains('table-warning') || tr.classList.contains('row-editing')) return;
            if (tr.closest('tbody').querySelector('tr.row-editing')) return;
            const row = this._rowDataMap.get(String(tr.dataset.id));
            if (row) this._enableRowEdit(tr, row);
        });

        this._initDataTable(table);
    }

    /** Crea un <tr> para un registro. Los eventos se manejan por delegación en el tbody. */
    _renderRow(row) {
        const tr = document.createElement('tr');
        tr.dataset.id = row.Id;

        // rowClassIf permite marcar filas segun datos (ej: ClaseEsNueva).
        if (typeof this.config.rowClassIf === 'function') {
            const extraCls = this.config.rowClassIf(row);
            if (extraCls) tr.classList.add(...String(extraCls).split(/\s+/).filter(Boolean));
        }

        // Claves que son secundarias en un grupo: no se renderizan como columna separada.
        const groupedSecondary = new Set();
        this.config.columns.forEach(c => { if (c.groupWith) groupedSecondary.add(c.groupWith); });

        this.config.columns.forEach(col => {
            if (col.visible === false) return;
            if (groupedSecondary.has(col.key)) return;
            const td = document.createElement('td');
            td.dataset.key = col.key;
            this._renderCellDisplay(td, row, col);
            tr.appendChild(td);
        });
        return tr;
    }

    /** Marca visualmente la fila seleccionada. */
    _selectRow(tr) {
        if (this.selectedRow) this.selectedRow.classList.remove('table-active');
        this.selectedRow = tr;
        tr.classList.add('table-active');
    }

    /** Muestra el valor de una celda en modo lectura. */
    _renderCellDisplay(td, row, col) {
        const type = col.input && col.input.type;

        // Si la columna agrupa con otra (ej: Precinto Interior + Exterior en la misma celda)
        if (col.groupWith) {
            const otherCol = this.config.columns.find(c => c.key === col.groupWith);
            const labels = col.groupLabels || ['', ''];
            const colors = col.groupColors || ['#2563eb', '#f59e0b']; // azul / ámbar por defecto
            const mainLabel = this._getDisplayValue(row, col);
            const otherLabel = otherCol ? this._getDisplayValue(row, otherCol) : '';
            td.innerHTML = '';
            const wrap = document.createElement('div');
            wrap.style.cssText = 'display:flex;flex-direction:column;align-items:stretch;gap:3px;line-height:1.15;';
            wrap.innerHTML =
                `<span style="display:inline-flex;align-items:center;gap:4px;">` +
                    `<b style="color:${colors[0]};font-size:.68rem;letter-spacing:.03em;min-width:26px;">${labels[0] || ''}</b>` +
                    `<span>${this._escapeHtml(mainLabel)}</span>` +
                `</span>` +
                `<span style="display:inline-flex;align-items:center;gap:4px;">` +
                    `<b style="color:${colors[1]};font-size:.68rem;letter-spacing:.03em;min-width:26px;">${labels[1] || ''}</b>` +
                    `<span>${this._escapeHtml(otherLabel)}</span>` +
                `</span>`;
            td.appendChild(wrap);
            return;
        }

        if (type === 'switch') {
            td.appendChild(this._createSwitchDisplay(!!row[col.key], false).wrap);
        } else if (type === 'checkbox') {
            td.appendChild(this._createStatusBadge(!!row[col.key], col.input.trueLabel, col.input.falseLabel));
        } else if (type === 'select' || type === 'autocomplete') {
            const opt = (col.input.options || []).find(o => String(o.value) === String(row[col.key]));
            td.textContent = opt ? opt.label : (row[col.displayKey] || '');
        } else if (type === 'date') {
            td.textContent = this._parseMsDate(row[col.key]);
        } else {
            // Si la columna tiene editableIf y la fila NO es editable, mostrar celda bloqueada.
            if (col.editableIf && !col.editableIf(row)) {
                td.innerHTML = '';
                const lockWrap = document.createElement('div');
                lockWrap.style.cssText = 'display:flex;align-items:center;gap:5px;opacity:.45;';
                const icon = document.createElement('span');
                icon.innerHTML = '';
                icon.style.cssText = 'font-size:.7rem;flex-shrink:0;';
                const txt = document.createElement('span');
                txt.style.cssText = 'font-size:.75rem;font-style:italic;color:#9ca3af;';
                txt.textContent = col.lockedTitle || 'No disponible';
                lockWrap.appendChild(icon);
                lockWrap.appendChild(txt);
                td.appendChild(lockWrap);
            } else {
                td.textContent = row[col.key] ?? '';
            }
        }
    }

    /** Obtiene el valor mostrable de una celda considerando displayKey y opciones. */
    _getDisplayValue(row, col) {
        const type = col.input && col.input.type;
        if (type === 'select' || type === 'autocomplete') {
            const opt = (col.input && col.input.options || []).find(o => String(o.value) === String(row[col.key]));
            if (opt) return opt.label;
            if (col.displayKey && row[col.displayKey]) return row[col.displayKey];
            return '';
        }
        return String(row[col.key] ?? '');
    }

    _escapeHtml(s) {
        if (s == null) return '';
        return String(s)
            .replace(/&/g, '&amp;').replace(/</g, '&lt;')
            .replace(/>/g, '&gt;').replace(/"/g, '&quot;');
    }

    /** Envuelve un input (wrap) con una etiqueta inline a la izquierda, en el color dado. */
    _wrapGroupedInput(innerWrap, label, color) {
        const container = document.createElement('div');
        container.style.cssText = 'display:flex;align-items:center;gap:6px;';
        const lbl = document.createElement('span');
        lbl.textContent = (label ? label + ':' : '');
        lbl.style.cssText = `color:${color};font-weight:700;font-size:.72rem;letter-spacing:.03em;min-width:26px;`;
        container.appendChild(lbl);
        innerWrap.style.flex = '1';
        container.appendChild(innerWrap);
        return container;
    }

    /** Destruye la instancia actual de DataTables y limpia el wrapper del DOM. */
    _destroyDataTable() {
        if (this.dataTableInstance) {
            try { this._savedPage = this.dataTableInstance.page(); } catch (_) { this._savedPage = 0; }
            try { this.dataTableInstance.destroy(); } catch (_) { }
            this.dataTableInstance = null;
        }
        const table = document.getElementById(this.config.tableId);
        if (table) {
            const wrapper = table.closest('.dataTables_wrapper');
            if (wrapper && wrapper.parentNode) {
                wrapper.parentNode.insertBefore(table, wrapper);
                wrapper.remove();
            }
        }
    }

    /** Calcula el alto disponible para el scroll de la tabla. */
    _calcScrollY(table) {
        const rect = table.getBoundingClientRect();
        const available = window.innerHeight - rect.top - 100;
        return Math.max(50, available) + 'px';
    }

    /** Activa el plugin jQuery DataTables con los botones de exportación. */
    _initDataTable(table) {
        try {
            if (typeof $ !== 'undefined' && $.fn && $.fn.DataTable) {
                this.dataTableInstance = $(table).DataTable({
                    columnDefs: [{
                        targets: '_all',
                        createdCell: cell => { $(cell).css({ 'vertical-align': 'middle', 'text-align': 'center', }); }
                    }],
                    headerCallback: thead => {
                        $(thead).find('th').addClass('text-center').css({ 'background': 'var(--main-section-border, #d71920)', 'color': '#fff' }); },
                    bSort: true, bInfo: true, paging: true, searching: true, order: [], dom: 'Bfrtip', pageLength: 15, scrollY: this._calcScrollY(table),
                    buttons: [
                        { extend: 'pdfHtml5', download: 'open', orientation: 'landscape', pageSize: 'LEGAL' },
                        { extend: 'excel' },
                        { extend: 'colvis', collectionLayout: 'fixed columns', text: 'Columnas', titleAttr: 'Column visibility control' }
                    ]
                });

                // Cualquier redibujado interno (paginar, ordenar, filtrar) repinta los locks.
                $(table).on('draw.dt', () => {
                    if (this.realTime) this.realTime.repaintAllLocks();
                });

                // Restaurar página previa si había una guardada.
                if (this._savedPage > 0) {
                    try { this.dataTableInstance.page(this._savedPage).draw(false); } catch (_) {}
                    this._savedPage = 0;
                }
            }
        } catch (ex) { console.error(ex); }
    }


    //////////////////// SECCIÓN 4 — Componentes de interfaz reutilizables ////////////////////
    // Badges, radio toggles, selects, text inputs,
    // autocomplete, switch y utilidad de fechas.

    /** Badge "Activo" (verde) / "Inactivo" (rojo). */
    _createStatusBadge(value, trueLabel, falseLabel) {
        const span = document.createElement('span');
        span.classList.add('badge', value ? 'bg-success' : 'bg-danger');
        span.textContent = value ? (trueLabel || 'Activo') : (falseLabel || 'Inactivo');
        const wrap = document.createElement('div');
        wrap.classList.add('checkbox-display');
        wrap.appendChild(span);
        return wrap;
    }

    /** Switch visual Si/No. Si editable=true, se puede hacer click para cambiar. */
    _createSwitchDisplay(value, editable) {
        const wrap  = document.createElement('div'); wrap.classList.add('switch-display');
        const track = document.createElement('div'); track.classList.add('switch-track');
        if (value)    track.classList.add('on');
        if (editable) track.classList.add('editable');
        const thumb = document.createElement('div'); thumb.classList.add('switch-thumb');
        track.appendChild(thumb);
        const label = document.createElement('span'); label.classList.add('switch-label');
        label.textContent = value ? 'Si' : 'No';
        wrap.appendChild(track); wrap.appendChild(label);

        let current = !!value;
        const refresh = () => { track.classList.toggle('on', current); label.textContent = current ? 'Si' : 'No'; };
        if (editable) {
            track.setAttribute('tabindex', '0');
            const toggle = () => { current = !current; refresh(); };
            track.addEventListener('click', toggle);
            track.addEventListener('keydown', e => { if (e.key === ' ' || e.key === 'Enter') { e.preventDefault(); toggle(); } });
        }
        return { wrap, track, getValue: () => current };
    }

    /** Botón toggle único que alterna entre dos estados al hacer click. */
    _createToggleButton(currentValue, trueLabel, falseLabel) {
        let value = !!currentValue;
        const tLabel = trueLabel || 'Activo';
        const fLabel = falseLabel || 'Inactivo';

        const btn = document.createElement('button');
        btn.type = 'button';
        btn.classList.add('btn', 'btn-sm');
        btn.tabIndex = 0;

        const sync = () => {
            btn.textContent = value ? tLabel : fLabel;
            btn.classList.remove(value ? 'btn-danger' : 'btn-success');
            btn.classList.add(value ? 'btn-success' : 'btn-danger');
        };
        sync();

        btn.addEventListener('click', (e) => { e.preventDefault(); value = !value; sync(); });
        btn.addEventListener('keydown', (e) => {
            if (e.key === ' ') { e.preventDefault(); value = !value; sync(); }
        });

        const wrap = document.createElement('div');
        wrap.classList.add('checkbox-display');
        wrap.appendChild(btn);

        return { wrap, btn, getValue: () => value };
    }

    /** Crea un <select> con las opciones dadas. */
    _createSelectInput(options, currentValue) {
        const input = document.createElement('select');
        input.classList.add('form-select', 'form-select-sm');
        (options || []).forEach(opt => {
            const o = document.createElement('option');
            o.value = opt.value; o.textContent = opt.label;
            if (String(opt.value) === String(currentValue)) o.selected = true;
            input.appendChild(o);
        });
        return input;
    }

    /** Crea un <input> de texto, número, fecha, email, etc. */
    _createTextInput(type, currentValue, cfg) {
        const input = document.createElement('input');
        input.type = type; input.value = currentValue ?? '';
        input.classList.add('form-control', 'form-control-sm');
        if (cfg) { if (cfg.min) input.min = cfg.min; if (cfg.max) input.max = cfg.max; }
        return input;
    }

    /**
     * Input con lista de sugerencias (autocomplete) para catálogos grandes.
     * Se escribe, se filtra la lista, se selecciona con click o teclado.
     */
    _createAutocompleteInput(options, currentValue, currentLabel, onSelect) {
        const wrap = document.createElement('div'); wrap.classList.add('autocomplete-wrapper');
        const input = document.createElement('input');
        input.type = 'text'; input.value = currentLabel ?? '';
        input.classList.add('form-control', 'form-control-sm'); input.autocomplete = 'off';

        // El dropdown se monta en <body> para escapar de overflow:hidden de DataTables.
        const list = document.createElement('div'); list.classList.add('autocomplete-list');
        document.body.appendChild(list);

        let selectedValue = currentValue ?? '';
        let activeIdx = -1;

        // Posicionar el dropdown justo debajo del input usando coordenadas fijas.
        const _positionList = () => {
            const rect = input.getBoundingClientRect();
            list.style.left   = rect.left + 'px';
            list.style.top    = (rect.bottom + 2) + 'px';
            list.style.width  = Math.max(rect.width, 180) + 'px';
        };

        const render = (filter) => {
            list.innerHTML = ''; activeIdx = -1;
            const term = (filter || '').toLowerCase();
            const filtered = (options || []).filter(o => !term || o.label.toLowerCase().includes(term));
            if (!filtered.length) { list.style.display = 'none'; return; }
            filtered.forEach(opt => {
                const div = document.createElement('div'); div.classList.add('ac-item'); div.textContent = opt.label;
                div.addEventListener('mousedown', e => {
                    e.preventDefault(); input.value = opt.label; selectedValue = opt.value;
                    list.style.display = 'none'; if (onSelect) onSelect(opt.value, opt.label);
                });
                list.appendChild(div);
            });
            _positionList();
            list.style.display = 'block';
        };

        const _hideList = () => { list.style.display = 'none'; };

        let _acTimer = null;
        input.addEventListener('input', () => { selectedValue = ''; clearTimeout(_acTimer); _acTimer = setTimeout(() => render(input.value), 80); });
        input.addEventListener('focus', () => { _positionList(); render(input.value); });
        input.addEventListener('blur',  () => { setTimeout(_hideList, 200); });

        // Limpiar el nodo del body cuando el wrapper se retire del DOM.
        const _observer = new MutationObserver(() => {
            if (!document.body.contains(wrap)) { list.remove(); _observer.disconnect(); }
        });
        _observer.observe(document.body, { childList: true, subtree: true });

        const resolveValue = () => {
            if (selectedValue !== '' && selectedValue != null) return selectedValue;
            const typed = (input.value || '').trim().toLowerCase();
            if (typed) { const match = (options || []).find(o => o.label.toLowerCase() === typed); if (match) { selectedValue = match.value; return match.value; } }
            return selectedValue;
        };

        input.addEventListener('keydown', e => {
            const items = list.querySelectorAll('.ac-item');
            if (e.key === 'ArrowDown')      { e.preventDefault(); activeIdx = Math.min(activeIdx + 1, items.length - 1); items.forEach((it, i) => it.classList.toggle('active', i === activeIdx)); if (items[activeIdx]) items[activeIdx].scrollIntoView({ block: 'nearest' }); }
            else if (e.key === 'ArrowUp')   { e.preventDefault(); activeIdx = Math.max(activeIdx - 1, 0); items.forEach((it, i) => it.classList.toggle('active', i === activeIdx)); if (items[activeIdx]) items[activeIdx].scrollIntoView({ block: 'nearest' }); }
            else if (e.key === 'Enter' && activeIdx >= 0 && items[activeIdx]) { e.preventDefault(); e.stopPropagation(); e.stopImmediatePropagation(); items[activeIdx].dispatchEvent(new Event('mousedown')); }
            else if (e.key === 'Tab') { if (activeIdx >= 0 && items[activeIdx]) items[activeIdx].dispatchEvent(new Event('mousedown')); else if (items.length === 1) items[0].dispatchEvent(new Event('mousedown')); }
        });

        wrap.appendChild(input);
        return { wrap, input, getValue: () => resolveValue(), getLabel: () => input.value, setValue: (val, label) => { selectedValue = val; input.value = label || ''; } };
    }

    /** Convierte fechas de .NET (/Date(ms)/) o ISO a formato yyyy-mm-dd. */
    _parseMsDate(raw) {
        if (!raw) return '';
        let d = null;
        if (typeof raw === 'string') {
            const match = raw.match(/\/Date\((-?\d+)\)\//);
            if (match) d = new Date(parseInt(match[1], 10));
            else if (/^\d{4}-\d{2}-\d{2}/.test(raw)) d = new Date(raw.slice(0, 10) + 'T00:00:00');
        }
        if (raw instanceof Date) d = raw;
        if (d && !isNaN(d.getTime()) && d.getFullYear() >= 1753) return d.toISOString().slice(0, 10);
        return '';
    }

    /**
     * Configura la resolución automática de un campo enlazado a partir del texto escrito.
     * Cuando el usuario escribe en un autocomplete y no selecciona una opción existente,
     * intenta resolver el campo destino usando fetchByTextUrl (ej. transportista por nomenclatura).
     */
    _setupLinkByText(inputEl, link, getTarget) {
        if (!link || !link.fetchByTextUrl) return;
        let _linkTimer = null;
        let _lastFetched = '';
        inputEl.addEventListener('input', () => {
            clearTimeout(_linkTimer);
            _linkTimer = setTimeout(async () => {
                const text = (inputEl.value || '').trim();
                if (!text || text === _lastFetched) return;
                _lastFetched = text;
                try {
                    const res = await fetch(link.fetchByTextUrl + '?codigo=' + encodeURIComponent(text));
                    const data = await res.json();
                    if (data && data.value) {
                        const target = getTarget();
                        if (target && target.setValue) target.setValue(data.value, data.label);
                    }
                } catch (_) { }
            }, 300);
        });
    }


    //////////////////// SECCIÓN 5 — Guardar, revertir y atajos de teclado ////////////////////
    // Manejo genérico de blur→guardar, Enter→guardar, Escape→cancelar.

    /**
     * Conecta los eventos de guardar/cancelar a un grupo de elementos editables.
     * Garantiza que solo se guarde una vez (evita doble disparo blur + Enter).
     */
    _attachSaveHandlers(elements, performSave, performCancel, container) {
        let saveInProgress = false;
        let savedOnce = false;
        const attachTime = Date.now();

        const guardedSave = async () => {
            if (saveInProgress || savedOnce) return;
            saveInProgress = true;
            try { await performSave(); savedOnce = true; }
            finally { saveInProgress = false; }
        };

        const blurHandler = async () => {
            if (saveInProgress || savedOnce) return;
            if (Date.now() - attachTime < 350) return;           // evitar disparo inmediato
            await new Promise(r => setTimeout(r, 250));           // esperar posible re-focus
            if (savedOnce || saveInProgress) return;
            if (container && container.contains(document.activeElement)) return;
            await guardedSave();
        };

        const keyHandler = async (e) => {
            if (e.key === 'Enter')       { e.preventDefault(); await guardedSave(); }
            else if (e.key === 'Escape') { e.preventDefault(); e.stopPropagation(); performCancel(); }
        };

        elements.forEach(el => { el.addEventListener('blur', blurHandler); el.addEventListener('keydown', keyHandler); });
    }

    /** Restaura una celda a su valor anterior (modo lectura). */
    _restoreCell(td, row, col, value) {
        row[col.key] = value;
        const type = col.input && col.input.type;

        if (type === 'switch') {
            td.innerHTML = ''; td.appendChild(this._createSwitchDisplay(!!value, false).wrap);
        } else if (type === 'checkbox') {
            td.innerHTML = ''; td.appendChild(this._createStatusBadge(value, col.input.trueLabel, col.input.falseLabel));
        } else if (type === 'select' || type === 'autocomplete') {
            const opt = (col.input.options || []).find(o => String(o.value) === String(value));
            td.textContent = opt ? opt.label : (row[col.displayKey] || value);
        } else if (type === 'date') {
            td.textContent = this._parseMsDate(value);
        } else {
            td.textContent = value;
        }
    }


    //////////////////// SECCIÓN 6 — Edición inline (doble clic) ////////////////////
    // Al hacer doble clic en una celda editable, se convierte
    // en input. Al guardar, se envía al servidor y se recarga.

    async _enableEdit(td, row, col) {
        if (td.querySelector('input, select')) return;

        /* Bloquear edición de campos que dependen de otro valor (ej. precintos ↔ Movimiento) */
        if (col.dependsOnMovimiento && row['Movimiento'] !== 'Internacional') return;

        const cfg      = col.input || { type: 'text' };
        const oldValue = row[col.key];

        const saveAndRefresh = async (newValue) => {
            row[col.key] = newValue;
            this._restoreCell(td, row, col, newValue);
            const result = await this._postRequest(this.config.urls.update, this._payloadToForm(this._buildPayload(row)));
            if (!result.ok) { alert(result.message); this._restoreCell(td, row, col, oldValue); return; }
            /* Solo recargar si no hay otra celda en edición */
            const table = document.getElementById(this.config.tableId);
            if (!table.querySelector('tbody input, tbody select')) {
                if (this.realTime) this.realTime.notifyDataChanged();
                await this._refresh();
            }
        };
        const cancel = () => this._restoreCell(td, row, col, oldValue);

        /* --- Select --- */
        if (cfg.type === 'select') {
            let options = cfg.options || [];
            const currentVal = row[col.key];
            if (currentVal != null && !options.some(o => String(o.value) === String(currentVal))) {
                const lbl = (col.displayKey && row[col.displayKey]) ? row[col.displayKey] : currentVal;
                options = [{ value: currentVal, label: lbl }, ...options];
            }
            const input = this._createSelectInput(options, currentVal);
            td.innerHTML = ''; td.appendChild(input); input.focus();
            this._attachSaveHandlers([input], () => saveAndRefresh(input.value), cancel);
            return;
        }

        /* --- Autocomplete --- */
        if (cfg.type === 'autocomplete') {
            const currentVal   = row[col.key];
            const currentLabel = (col.displayKey && row[col.displayKey]) ? row[col.displayKey] : '';
            let options = cfg.options || [];
            if (currentVal != null && !options.some(o => String(o.value) === String(currentVal)))
                options = [{ value: currentVal, label: currentLabel || currentVal }, ...options];
            const ac = this._createAutocompleteInput(options, currentVal, currentLabel, async (value) => {
                if (cfg.link) {
                    try {
                        const res = await fetch(cfg.link.fetchUrl + '?id=' + value);
                        const data = await res.json();
                        if (data && data.value) {
                            row[cfg.link.targetKey] = data.value;
                            const linkCol = this.config.columns.find(c => c.key === cfg.link.targetKey);
                            if (linkCol && linkCol.displayKey) row[linkCol.displayKey] = data.label;
                        }
                    } catch (_) { }
                }
            });
            if (cfg.link) this._setupLinkByText(ac.input, cfg.link, () => ({
                setValue: (val, label) => {
                    row[cfg.link.targetKey] = val;
                    const linkCol = this.config.columns.find(c => c.key === cfg.link.targetKey);
                    if (linkCol && linkCol.displayKey) row[linkCol.displayKey] = label;
                }
            }));
            td.innerHTML = ''; td.appendChild(ac.wrap); ac.input.focus();
            this._attachSaveHandlers([ac.input], () => {
                if (col.allowNewEntry && col.displayKey) {
                    row[col.displayKey] = ac.getLabel();
                }
                return saveAndRefresh(ac.getValue());
            }, cancel, ac.wrap);
            return;
        }

        /* --- Switch (Si / No) --- */
        if (cfg.type === 'switch') {
            const sw = this._createSwitchDisplay(!!row[col.key], true);
            td.innerHTML = ''; td.appendChild(sw.wrap); sw.track.focus();
            this._attachSaveHandlers([sw.track], () => saveAndRefresh(sw.getValue()), cancel, td);
            return;
        }

        /* --- Checkbox (Activo / Inactivo) --- */
        if (cfg.type === 'checkbox') {
            const toggle = this._createToggleButton(row[col.key], cfg.trueLabel, cfg.falseLabel);
            td.innerHTML = ''; td.appendChild(toggle.wrap); toggle.btn.focus();
            this._attachSaveHandlers([toggle.btn], () => saveAndRefresh(toggle.getValue()), cancel, td);
            return;
        }

        /* --- Texto, fecha, número, email, etc. --- */
        const editValue = cfg.type === 'date' ? this._parseMsDate(row[col.key]) : row[col.key];
        const input = this._createTextInput(cfg.type, editValue, cfg);
        td.innerHTML = ''; td.appendChild(input); input.focus();
        this._attachSaveHandlers([input], () => saveAndRefresh(input.value), cancel);
    }


    //////////////////// SECCIÓN 6.5 — Edición de fila completa (doble clic / F2) ////////////////////
    // Pone toda la fila en modo edición. El usuario decide qué campos modificar.
    // Permanece en modo edición hasta que pulse Enter (guardar) o Escape (cancelar).

    async _enableRowEdit(tr, row) {
        if (tr.classList.contains('row-editing')) return;

        // Rechazar edición si otro usuario ya está editando este registro.
        if (this.realTime && this.realTime.isRowLockedByOther(row.Id)) return;

        const editableCols = this.config.columns.filter(c => c.editable && (c.visible !== false || (c.editableIf && c.editableIf(row))));        if (!editableCols.length) return;

        tr.classList.add('row-editing');

        // Notificar a otros usuarios que este registro está siendo editado.
        if (this.realTime) this.realTime.notifyEditing(row.Id);

        const oldValues = {};
        const inputs = {};
        const inputElements = [];
        const hiddenCellKeys = new Set(); // celdas que estaban ocultas y se mostraron durante edición
        const tempCells = [];            // celdas creadas dinámicamente, se eliminan al terminar

        // Mapa de columnas secundarias -> principal (para groupWith)
        const groupedSecondaryCols = new Set();
        editableCols.forEach(c => { if (c.groupWith) groupedSecondaryCols.add(c.groupWith); });

        for (const col of editableCols) {
            // Si esta columna es la secundaria de un grupo, se renderiza dentro de su columna principal (no aquí).
            if (groupedSecondaryCols.has(col.key)) continue;

            let td = tr.querySelector(`td[data-key="${col.key}"]`);
            // Si la columna es visible:false con editableIf, crear un td flotante fuera del DOM
            // para alojar el input sin afectar el conteo de columnas de DataTables.
            let isTempCell = false;
            if (!td && col.editableIf) {
                td = document.createElement('td');
                td.dataset.key = col.key;
                td.style.cssText = 'position:fixed;visibility:hidden;pointer-events:none;width:0;height:0;overflow:hidden;';
                document.body.appendChild(td);
                tempCells.push(td);
                isTempCell = true;
            }
            if (!td) continue;

            // Si la celda estaba oculta (visible:false con editableIf), mostrarla durante edición.
            const wasHidden = td.style.display === 'none';
            if (wasHidden) { td.style.display = ''; hiddenCellKeys.add(col.key); }

            if (col.dependsOnMovimiento && row['Movimiento'] !== 'Internacional') continue;
            if (td.querySelector('input, select')) continue;

            // editableIf permite bloquear el campo para filas especificas (ej: BienTrans solo si la Clase es nueva).
            const isEditable = !col.editableIf || col.editableIf(row);
            if (!isEditable) {
                const lockedInput = document.createElement('input');
                lockedInput.type = 'text';
                lockedInput.classList.add('form-control', 'form-control-sm');
                lockedInput.value = '';
                lockedInput.disabled = true;
                lockedInput.placeholder = (col.lockedTitle || 'No disponible');
                lockedInput.style.cssText = 'background:#f3f4f6;color:#9ca3af;cursor:not-allowed;border-color:#e5e7eb;font-style:italic;font-size:.75rem;';
                td.innerHTML = ''; td.appendChild(lockedInput);
                inputs[col.key] = { getValue: () => row[col.key] };
                continue;
            }

            oldValues[col.key] = row[col.key];
            if (col.displayKey) oldValues[col.displayKey] = row[col.displayKey];

            const cfg = col.input || { type: 'text' };

            /* --- Select --- */
            if (cfg.type === 'select') {
                let options = cfg.options || [];
                const currentVal = row[col.key];
                if (currentVal != null && !options.some(o => String(o.value) === String(currentVal))) {
                    const lbl = (col.displayKey && row[col.displayKey]) ? row[col.displayKey] : currentVal;
                    options = [{ value: currentVal, label: lbl }, ...options];
                }
                const input = this._createSelectInput(options, currentVal);
                td.innerHTML = ''; td.appendChild(input);
                inputs[col.key] = { getValue: () => input.value };
                inputElements.push(input);
            }
            /* --- Autocomplete --- */
            else if (cfg.type === 'autocomplete') {
                const currentVal   = row[col.key];
                const currentLabel = (col.displayKey && row[col.displayKey]) ? row[col.displayKey] : '';
                let options = cfg.options || [];
                if (currentVal != null && !options.some(o => String(o.value) === String(currentVal)))
                    options = [{ value: currentVal, label: currentLabel || currentVal }, ...options];
                const ac = this._createAutocompleteInput(options, currentVal, currentLabel, async (value) => {
                    if (cfg.link) {
                        try {
                            const res = await fetch(cfg.link.fetchUrl + '?id=' + value);
                            const data = await res.json();
                            if (data && data.value) {
                                row[cfg.link.targetKey] = data.value;
                                const linkCol = this.config.columns.find(c => c.key === cfg.link.targetKey);
                                if (linkCol && linkCol.displayKey) row[linkCol.displayKey] = data.label;
                            }
                        } catch (_) { }
                    }
                });
                if (cfg.link) this._setupLinkByText(ac.input, cfg.link, () => {
                    const targetKey = cfg.link.targetKey;
                    return {
                        setValue: (val, label) => {
                            row[targetKey] = val;
                            const linkCol = this.config.columns.find(c => c.key === targetKey);
                            if (linkCol && linkCol.displayKey) row[linkCol.displayKey] = label;
                        }
                    };
                });
                td.innerHTML = ''; td.appendChild(ac.wrap);
                inputs[col.key] = { getValue: () => ac.getValue(), getLabel: () => ac.getLabel(), col };
                inputElements.push(ac.input);

                // Si esta columna agrupa con otra, crear el segundo input apilado en la misma celda.
                if (col.groupWith) {
                    const secCol = this.config.columns.find(c => c.key === col.groupWith);
                    if (secCol && secCol.editable && secCol.input && secCol.input.type === 'autocomplete') {
                        oldValues[secCol.key] = row[secCol.key];
                        if (secCol.displayKey) oldValues[secCol.displayKey] = row[secCol.displayKey];
                        const sCfg = secCol.input;
                        const sVal = row[secCol.key];
                        const sLbl = (secCol.displayKey && row[secCol.displayKey]) ? row[secCol.displayKey] : '';
                        let sOpts = sCfg.options || [];
                        if (sVal != null && !sOpts.some(o => String(o.value) === String(sVal))) {
                            sOpts = [{ value: sVal, label: sLbl || sVal }, ...sOpts];
                        }

                        // Envolver el autocomplete principal con su etiqueta (Int:)
                        const labels = col.groupLabels || ['', ''];
                        const colors = col.groupColors || ['#2563eb', '#f59e0b'];
                        const mainWrap = this._wrapGroupedInput(ac.wrap, labels[0], colors[0]);

                        // Reemplazar el contenido del td: etiquetado primario + etiquetado secundario
                        td.innerHTML = '';
                        td.appendChild(mainWrap);

                        const sac = this._createAutocompleteInput(sOpts, sVal, sLbl);
                        const secWrap = this._wrapGroupedInput(sac.wrap, labels[1], colors[1]);
                        secWrap.style.marginTop = '4px';
                        td.appendChild(secWrap);

                        inputs[secCol.key] = { getValue: () => sac.getValue(), getLabel: () => sac.getLabel(), col: secCol };
                        inputElements.push(sac.input);
                    }
                }
            }
            /* --- Switch --- */
            else if (cfg.type === 'switch') {
                const sw = this._createSwitchDisplay(!!row[col.key], true);
                td.innerHTML = ''; td.appendChild(sw.wrap);
                inputs[col.key] = { getValue: () => sw.getValue() };
                inputElements.push(sw.track);
            }
            /* --- Checkbox --- */
            else if (cfg.type === 'checkbox') {
                const toggle = this._createToggleButton(row[col.key], cfg.trueLabel, cfg.falseLabel);
                td.innerHTML = ''; td.appendChild(toggle.wrap);
                inputs[col.key] = { getValue: () => toggle.getValue() };
                inputElements.push(toggle.btn);
            }
            /* --- Texto, fecha, número, email, etc. --- */
            else {
                const editValue = cfg.type === 'date' ? this._parseMsDate(row[col.key]) : row[col.key];
                const input = this._createTextInput(cfg.type, editValue, cfg);
                td.innerHTML = ''; td.appendChild(input);
                inputs[col.key] = { getValue: () => input.value };
                inputElements.push(input);
            }
        }

        if (inputElements.length > 0) inputElements[0].focus();

        let saveInProgress = false;

        const cancelAll = () => {
            for (const col of editableCols) {
                if (oldValues[col.key] === undefined) continue;
                row[col.key] = oldValues[col.key];
                if (col.displayKey && oldValues[col.displayKey] !== undefined) row[col.displayKey] = oldValues[col.displayKey];
                const td = tr.querySelector(`td[data-key="${col.key}"]`);
                if (td) {
                    td.innerHTML = '';
                    this._renderCellDisplay(td, row, col);
                    if (hiddenCellKeys.has(col.key)) td.style.display = 'none';
                }
            }
            tempCells.forEach(td => td.remove());
            tr.classList.remove('row-editing');
            if (this.realTime) this.realTime.notifyDoneEditing(row.Id);
            cleanup();
            this.flushRemoteRefresh();
        };

        const saveAll = async () => {
            if (saveInProgress) return;
            saveInProgress = true;
            try {
                for (const col of editableCols) {
                    const inp = inputs[col.key];
                    if (!inp) continue;
                    row[col.key] = inp.getValue();
                    if (col.allowNewEntry && col.displayKey && inp.getLabel) {
                        row[col.displayKey] = inp.getLabel();
                    }
                }

                for (const col of editableCols) {
                    const td = tr.querySelector(`td[data-key="${col.key}"]`);
                    if (td) {
                        td.innerHTML = '';
                        this._renderCellDisplay(td, row, col);
                        if (hiddenCellKeys.has(col.key)) td.style.display = 'none';
                    }
                }
                tempCells.forEach(td => td.remove());
                tr.classList.remove('row-editing');
                cleanup();

                const result = await this._postRequest(this.config.urls.update, this._payloadToForm(this._buildPayload(row)));
                if (!result.ok) {
                    alert(result.message);
                    for (const col of editableCols) {
                        if (oldValues[col.key] === undefined) continue;
                        row[col.key] = oldValues[col.key];
                        if (col.displayKey && oldValues[col.displayKey] !== undefined) row[col.displayKey] = oldValues[col.displayKey];
                        const td = tr.querySelector(`td[data-key="${col.key}"]`);
                        if (td) {
                            td.innerHTML = '';
                            this._renderCellDisplay(td, row, col);
                            if (hiddenCellKeys.has(col.key)) td.style.display = 'none';
                        }
                    }
                    if (this.realTime) this.realTime.notifyDoneEditing(row.Id);
                    this.flushRemoteRefresh();
                    return;
                }
                if (this.realTime) {
                    this.realTime.notifyDoneEditing(row.Id);
                    this.realTime.notifyDataChanged();
                }
                await this._refresh();
            } finally { saveInProgress = false; }
        };

        const keyHandler = async (e) => {
            if (e.key === 'Enter') {
                e.preventDefault();
                await saveAll();
            } else if (e.key === 'Escape') {
                e.preventDefault();
                e.stopPropagation();
                cancelAll();
            }
        };

        const clickOutsideHandler = async (e) => {
            if (saveInProgress) return;
            if (tr.contains(e.target)) return;
            /* Ignorar clics en dropdowns de autocomplete que flotan fuera del tr */
            if (e.target.closest && e.target.closest('.autocomplete-list')) return;
            /* Dar tiempo a que el foco se asiente antes de decidir */
            await new Promise(r => setTimeout(r, 100));
            if (saveInProgress) return;
            if (tr.contains(document.activeElement)) return;
            await saveAll();
        };

        const cleanup = () => {
            inputElements.forEach(el => el.removeEventListener('keydown', keyHandler));
            document.removeEventListener('mousedown', clickOutsideHandler, true);
        };

        inputElements.forEach(el => el.addEventListener('keydown', keyHandler));
        document.addEventListener('mousedown', clickOutsideHandler, true);
    }


    //////////////////// SECCI0N 7 — Agregar fila nueva ////////////////////



    // pPra capturar un nuevo registro. Enter guarda, Escape cancela, para corregir el bug
    addRow() {
        /* Límite de registros en la tabla */
        if (this.config.maxActiveRows) {
            const count = this.tableData.length;
            if (count >= this.config.maxActiveRows) {
                alert(`No se pueden agregar más registros. Ya existen ${count} registros (máximo ${this.config.maxActiveRows}).`);
                return;
            }
        }

        if (this.creatingNewRow) {
            const table = document.getElementById(this.config.tableId);
            if (!table || !table.querySelector('tr.table-warning')) {
                this.creatingNewRow = false;
            } else {
                return;
            }
        }
        this.creatingNewRow = true;
        if (this.realTime) this.realTime.notifyAdding();

        const table = document.getElementById(this.config.tableId);
        const tbody = table.querySelector('tbody') || document.createElement('tbody');
        const tr = document.createElement('tr'); tr.classList.add('table-warning');

        const inputs = {};
        const inputsOrder = [];
        let tabIndex = 1;

        // Columnas que son secundarias de un groupWith: no se renderizan como celda propia.
        const groupedSecondary = new Set();
        this.config.columns.forEach(c => { if (c.groupWith) groupedSecondary.add(c.groupWith); });

        this.config.columns.forEach(col => {
            if (col.visible === false) return;
            if (groupedSecondary.has(col.key)) return;
            const td = document.createElement('td');

            /* Columnas que no se editan en la fila nueva → celda vacía */
            if (col.key === 'Id')     { td.textContent = ''; tr.appendChild(td); return; }
            if (!col.persist)         { td.textContent = ''; tr.appendChild(td); return; }
            if (col.defaultValue !== undefined && !col.input && !col.editable) {
                td.textContent = col.defaultValue; tr.appendChild(td); return;
            }
            /* Campo oculto enlazado (ej. Load_Id llenado por un autocomplete) */
            if (!col.input && !col.editable && col.persist && col.key !== 'Id') {
                let hiddenValue = '';
                td.textContent = '';
                inputs[col.key] = { getValue: () => hiddenValue, setValue: (val, label) => { hiddenValue = val; td.textContent = label || val; } };
                tr.appendChild(td); return;
            }

            /* --- Switch --- */
            if (col.input && col.input.type === 'switch') {
                const sw = this._createSwitchDisplay(col.defaultValue ?? false, true);
                td.appendChild(sw.wrap); inputs[col.key] = sw; inputsOrder.push(sw.track);
            }
            /* --- Checkbox --- */
            else if (col.input && col.input.type === 'checkbox') {
                const toggle = this._createToggleButton(col.defaultValue ?? false, col.input.trueLabel, col.input.falseLabel);
                td.appendChild(toggle.wrap); inputs[col.key] = toggle; inputsOrder.push(toggle.btn);
            }
            /* --- Autocomplete --- */
            else if (col.input && col.input.type === 'autocomplete') {
                const colRef = col;
                const ac = this._createAutocompleteInput(col.input.options, null, '', async (value) => {
                    if (colRef.input.link && inputs[colRef.input.link.targetKey]) {
                        try {
                            const res = await fetch(colRef.input.link.fetchUrl + '?id=' + value);
                            const data = await res.json();
                            if (data && data.value) { const target = inputs[colRef.input.link.targetKey]; if (target.setValue) target.setValue(data.value, data.label); }
                        } catch (_) { }
                    }
                });
                if (colRef.input.link) this._setupLinkByText(ac.input, colRef.input.link, () => inputs[colRef.input.link.targetKey]);
                ac.input.tabIndex = tabIndex++; td.appendChild(ac.wrap); inputs[col.key] = ac; inputsOrder.push(ac.input);

                // Si esta columna agrupa con otra, apilar el input secundario en la misma celda.
                if (col.groupWith) {
                    const secCol = this.config.columns.find(c => c.key === col.groupWith);
                    if (secCol && secCol.input && secCol.input.type === 'autocomplete') {
                        const labels = col.groupLabels || ['', ''];
                        const colors = col.groupColors || ['#2563eb', '#f59e0b'];

                        // Re-envolver el autocomplete principal con etiqueta Int:
                        const mainWrap = this._wrapGroupedInput(ac.wrap, labels[0], colors[0]);
                        td.innerHTML = '';
                        td.appendChild(mainWrap);

                        const sac = this._createAutocompleteInput(secCol.input.options, null, '');
                        const secWrap = this._wrapGroupedInput(sac.wrap, labels[1], colors[1]);
                        secWrap.style.marginTop = '4px';
                        sac.input.tabIndex = tabIndex++;
                        td.appendChild(secWrap);
                        inputs[secCol.key] = sac;
                        inputsOrder.push(sac.input);
                    }
                }
            }
            /* --- Select --- */
            else if (col.input && col.input.type === 'select') {
                const sel = this._createSelectInput(col.input.options, null);
                sel.tabIndex = tabIndex++; td.appendChild(sel); inputs[col.key] = sel; inputsOrder.push(sel);
            }
            /* --- Texto, fecha, número, email, etc. --- */
            else {
                const input = this._createTextInput((col.input && col.input.type) || 'text', col.defaultValue ?? '', col.input);
                input.tabIndex = tabIndex++; td.appendChild(input); inputs[col.key] = input; inputsOrder.push(input);
            }

            tr.appendChild(td);
        });

        /* Dependencias entre campos (ej. precintos ↔ Movimiento) */
        const movSelect = inputs['Movimiento'];
        if (movSelect && movSelect.tagName) {
            const precintoCols = this.config.columns.filter(c => c.dependsOnMovimiento);
            const syncState = () => {
                const isInternacional = movSelect.value === 'Internacional';
                precintoCols.forEach(col => {
                    const inp = inputs[col.key]; if (!inp) return;
                    const el = inp.input || inp; const wrap = inp.wrap || el.parentElement;
                    if (isInternacional) { if (el.disabled !== undefined) el.disabled = false; if (wrap) wrap.style.opacity = '1'; }
                    else { if (el.disabled !== undefined) el.disabled = true; if (wrap) wrap.style.opacity = '0.4'; if (inp.input) inp.input.value = ''; }
                });
            };
            movSelect.addEventListener('change', syncState);
            syncState();
        }

        if (tbody.firstChild) tbody.insertBefore(tr, tbody.firstChild); else tbody.appendChild(tr);
        if (!table.querySelector('tbody')) table.appendChild(tbody);

        // que nuestro focus/target/inicio o pues comienzo sea en el primer elemento de nuestra lista.
        tr.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
        if (inputsOrder.length > 0) inputsOrder[0].focus();

        const removeNewRow = () => {
            try { window.removeEventListener('keydown', escHandler); } catch (_) { }
            tr.remove();
            this.creatingNewRow = false;
            if (this.realTime) this.realTime.notifyDoneAdding();
            this.flushRemoteRefresh();
        };
        const escHandler = (e) => { if (e.key === 'Escape') { e.preventDefault(); removeNewRow(); } };
        window.addEventListener('keydown', escHandler);

        const saveNew = async () => {
            const payload = this._collectNewRowPayload(inputs);

            // Validar campos obligatorios 
            const requiredCols = this.config.columns.filter(c => c.required);
            for (const col of requiredCols) {
                const val = payload[col.key] === undefined || payload[col.key] === null ? '' : String(payload[col.key]).trim();
                    if (!val.length) {
                        if (col.allowNewEntry && col.displayKey && payload[col.displayKey] && String(payload[col.displayKey]).trim().length) continue;
                        alert(`El campo "${col.label}" es obligatorio.`); return false;
                    }
            }

            /* Hook personalizado de pre-inserción (ej. verificar entries disponibles) */
            if (this.config.hooks && this.config.hooks.beforeInsert) {
                const allowed = await this.config.hooks.beforeInsert(payload);
                if (!allowed) return false;
            }

            const result = await this._postRequest(this.config.urls.insert, this._payloadToForm(payload));
            if (!result.ok) { alert(result.message); return false; }

            removeNewRow();
            if (this.realTime) this.realTime.notifyDataChanged();
            await this._refresh();
            return true;
        };

        inputsOrder.forEach(inp => {
            const handler = async e => {
                if (e.key === 'Enter')       { e.preventDefault(); await saveNew(); }
                else if (e.key === 'Escape') { e.preventDefault(); removeNewRow(); }
            };
            inp.addEventListener('keydown', handler);
            if (inp.type === 'radio' && inp.labels && inp.labels.length)
                Array.from(inp.labels).forEach(lbl => lbl.addEventListener('keydown', handler));
        });
    }

    /** Lee los valores de todos los inputs de la fila nueva y arma el payload. */
    _collectNewRowPayload(inputs) {
        const payload = {};
        for (const col of this.config.columns) {
            if (!col.persist) continue;
            const inp = inputs[col.key];
            if (!inp) { if (col.defaultValue !== undefined) payload[col.key] = col.defaultValue; continue; }

            const tag = inp.tagName && inp.tagName.toLowerCase();
            if (tag === 'select')                                       { payload[col.key] = inp.value; }
            else if (inp.getValue && typeof inp.getValue === 'function') {
                payload[col.key] = inp.getValue();
                if (col.allowNewEntry && col.displayKey && inp.getLabel && typeof inp.getLabel === 'function') {
                    payload[col.displayKey] = inp.getLabel();
                }
            }
            else if (tag === 'input') {
                const itype = (inp.type || '').toLowerCase();
                if (itype === 'checkbox')    payload[col.key] = inp.checked;
                else                         payload[col.key] = inp.value.trim();
            }
            else { try { payload[col.key] = inp.value ? inp.value.trim() : ''; } catch (_) { payload[col.key] = ''; } }
        }
        return payload;
    }


    //////////////////// SECCIÓN 8 — Eliminar fila ////////////////////
    // Borra el registro seleccionado (click previo) vía POST.

    async deleteRow() {
        if (!this.selectedRow) return;
        const row = this.selectedRow;
        const id  = row.dataset.id;

        // Mostrar puntitos rojos en todas las celdas mientras se espera respuesta
        const cells = Array.from(row.querySelectorAll('td'));
        const origContents = cells.map(td => td.innerHTML);
        cells.forEach((td, i) => {
            if (i === 0) {
                td.innerHTML = '<span class="ct-deleting-dots"><span></span><span></span><span></span></span>';
            } else {
                td.innerHTML = '';
            }
        });
        row.style.background = '#fff5f5';
        row.style.pointerEvents = 'none';

        const result = await this._postRequest(
            this.config.urls.delete,
            new URLSearchParams({ id: id.toString() })
        );

        if (!result.ok) {
            // Restaurar fila si hubo error
            cells.forEach((td, i) => td.innerHTML = origContents[i]);
            row.style.background = '';
            row.style.pointerEvents = '';
            alert(result.message);
            return;
        }

        // Éxito — fade-out y remove directo sin _refresh()
        row.classList.add('ct-row-removing');
        setTimeout(() => row.remove(), 370);

        this.selectedRow = null;
        this._rowDataMap.delete(String(id));
        if (this.realTime) this.realTime.notifyDataChanged();
    }




    //////////////////// SECCIÓN 9 — Shortcuts de teclado ////////////////////

    _initShortcuts() {
        if (this._shortcutsInitialized) return;
        this._shortcutsInitialized = true;

        this._shortcutHandler = async (e) => {

            const active = document.activeElement;
            const tag = active && active.tagName.toLowerCase();
            const isTyping = tag === 'input' || tag === 'select' || tag === 'textarea';
            const table = document.getElementById(this.config.tableId);
            const wrapper = table && (table.closest('.dataTables_wrapper') || table.closest('.card'));

            // ── Agregar registros ──

            // Alt + N — Nuevo registro
            if (e.altKey && e.key === 'n') {
                e.preventDefault();
                if (!this.creatingNewRow) this.addRow();
                return;
            }

            // Ctrl + S — Guardar registro actual
            if (e.ctrlKey && !e.shiftKey && e.key === 's') {
                e.preventDefault();
                if (isTyping && table && table.contains(document.activeElement)) {
                    document.activeElement.dispatchEvent(new KeyboardEvent('keydown', { key: 'Enter', bubbles: true }));
                }
                return;
            }

            // Ctrl + Enter — Guardar y crear otro registro nuevo
            if (e.ctrlKey && e.key === 'Enter' && isTyping) {
                if (table && table.contains(document.activeElement)) {
                    this._addRowAfterSave = true;
                }
                return;
            }

            // ── Eliminar y editar ──

            // Del — Eliminar fila seleccionada
            if (e.key === 'Delete' && !isTyping) {
                e.preventDefault();
                if (this.selectedRow) {
                    const ok = window.confirm('¿Deseas eliminar el registro seleccionado?');
                    if (ok) await this.deleteRow();
                }
                return;
            }

            // F2 — Editar todo el registro seleccionado
            if (e.key === 'F2' && !isTyping) {
                e.preventDefault();
                if (!this.selectedRow) return;
                if (this.selectedRow.classList.contains('row-editing')) return;
                const tableEl = document.getElementById(this.config.tableId);
                if (tableEl && tableEl.querySelector('tr.row-editing')) return;
                const id = this.selectedRow.dataset.id;
                const row = this._rowDataMap.get(String(id));
                if (!row) return;
                this._enableRowEdit(this.selectedRow, row);
                return;
            }

            // ── Tablas y listas ──

            // ↑ / ↓ — Navegar entre filas
            if (!isTyping && (e.key === 'ArrowDown' || e.key === 'ArrowUp')) {
                e.preventDefault();
                const tbody = table && table.querySelector('tbody');
                if (!tbody) return;
                const rows = Array.from(tbody.querySelectorAll('tr:not(.table-warning)'));
                if (!rows.length) return;
                let idx = this.selectedRow ? rows.indexOf(this.selectedRow) : -1;
                idx = e.key === 'ArrowDown' ? Math.min(idx + 1, rows.length - 1) : Math.max(idx - 1, 0);
                this._selectRow(rows[idx]);
                rows[idx].scrollIntoView({ block: 'nearest' });
                return;
            }

            // Ctrl + F — Enfocar buscador (Search:) de la DataTable
            if (e.ctrlKey && !e.shiftKey && e.key === 'f') {
                e.preventDefault();
                const searchInput = wrapper && wrapper.querySelector('.dataTables_filter input');
                if (searchInput) {
                    searchInput.focus();
                    searchInput.select();
                }
                return;
            }

            // Ctrl + E — Exportar tabla a Excel
            if (e.ctrlKey && e.key === 'e') {
                e.preventDefault();
                const excelBtn = wrapper && wrapper.querySelector('.buttons-excel');
                if (excelBtn) excelBtn.click();
                return;
            }

            // Ctrl + Shift + H — Ir al primer registro
            if (e.ctrlKey && e.shiftKey && e.key === 'H') {
                e.preventDefault();
                const tbody = table && table.querySelector('tbody');
                if (!tbody) return;
                const rows = Array.from(tbody.querySelectorAll('tr:not(.table-warning)'));
                if (rows.length) { this._selectRow(rows[0]); rows[0].scrollIntoView({ block: 'nearest' }); }
                return;
            }

            // Ctrl + Shift + E — Ir al último registro
            if (e.ctrlKey && e.shiftKey && e.key === 'E') {
                e.preventDefault();
                const tbody = table && table.querySelector('tbody');
                if (!tbody) return;
                const rows = Array.from(tbody.querySelectorAll('tr:not(.table-warning)'));
                if (rows.length) { const last = rows[rows.length - 1]; this._selectRow(last); last.scrollIntoView({ block: 'nearest' }); }
                return;
            }
        };
        window.addEventListener('keydown', this._shortcutHandler);
    }


    //////////////////// Botones de la barra superior ////////////////////

    /** Conecta los botones "Agregar" y "Eliminar" del card-header (solo una vez). */
    _attachButtons() {
        const btnAdd    = document.getElementById(this.config.btnAddId || 'btnAdd');
        const btnDelete = document.getElementById(this.config.btnDeleteId || 'btnDelete');
        if (btnAdd && !btnAdd._crudBound)    { btnAdd.addEventListener('click', () => this.addRow());    btnAdd._crudBound = true; }
        if (btnDelete && !btnDelete._crudBound) { btnDelete.addEventListener('click', () => this.deleteRow()); btnDelete._crudBound = true; }
    }
}
