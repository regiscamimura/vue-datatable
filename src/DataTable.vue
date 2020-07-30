<template>
	<div class="datatable-wrapper" ref="tableWrapper">
		<div ref="print-header" style="display: none" class="print-header">
			<slot name="print-header"></slot>
		</div>
		<div class="table-header">
			<span class="table-title">{{ title }}</span>
			<div class="actions">
				<a v-for="(button, index) in customButtons" v-if="button.hide ? !button.hide : true"
					:key="index"
					href="javascript:undefined"
					class="waves-effect btn-flat nopadding"
					@click="button.onclick"
				>
					<i class="material-icons">{{ button.icon }}</i>
				</a>
				<a v-if="printable"
					href="javascript:undefined"
					class="waves-effect btn-flat nopadding"
					@click="print"
				>
					<i class="material-icons">print</i>
				</a>
				<a v-if="exportable"
					href="javascript:undefined"
					class="waves-effect btn-flat nopadding"
					@click="exportExcel"
				>
					<i class="material-icons">description</i>
				</a>
				<a v-if="searchable"
					href="javascript:undefined"
					class="waves-effect btn-flat nopadding"
					@click="search"
				>
					<i class="material-icons">search</i>
				</a>
			</div>
		</div>
		<div v-if="searching">
			<div id="search-input-container">
				<label>
					<input id="search-input" type="search" class="form-control" :placeholder="lang['search_data']"
						v-model="searchInput"
					>
				</label>
			</div>
		</div>
		<table ref="table" :class="className">
			<thead>
				<tr>
					<th v-for="(column, index) in columns"
						:key="index"
						:class="(sortable ? 'sorting ' : '')
							+ (sortColumn === index ?
								(sortType === 'desc' ? 'sorting-desc' : 'sorting-asc')
								: '')
							+ (column.numeric ? ' numeric' : '')"
						:style="{width: column.width ? column.width : 'auto'}"
						@click="sort(index)"
					>
						{{ column.label }}
					</th>
					<slot name="thead-tr" />
				</tr>
			</thead>

			<tbody>
				<tr v-for="(row, index) in paginated"
					:key="index"
					:class="{ clickable : clickable }"
					@click="click(row)"
				>
					<td v-for="(column, columnIndex) in columns"
						:key="columnIndex"
						:class="{ numeric : column.numeric }"
					>
						<div v-if="!column.html">
							{{ collect(row, column.field) }}
						</div>
						<div v-if="column.html" v-html="collect(row, column.field)" />
					</td>
					<slot name="tbody-tr" :row="row" />
				</tr>

				<template v-if="rows.length === 0 && loadingAnimation === true">
					<tr v-for="n in (currentPerPage === -1 ? 10 : currentPerPage)"
						:key="n"
					>
						<td :colspan="columns.length">
							<tb-skeleton :height="15" theme="opacity" bg-color="#dcdbdc" shape="radius" />
						</td>
					</tr>
				</template>
			</tbody>
		</table>

		<div v-if="paginate" class="table-footer">
			<div class="buttons" :class="{'disabled': currentPage == 1}">
				<a href="javascript:undefined" @click.prevent="firstPage"> << </a>
				<a href="javascript:undefined" @click.prevent="previousPage"> < </a>
			</div>
			<div :class="{'datatable-length': true, 'rtl': lang.__is_rtl}">

				<div class="datatable-page-link">
					<a :class="{'selected': currentPage == n}" href="javascript:undefined" @click.prevent="goToPage(n)" v-for="n in Math.ceil(processedRows.length / currentPerPage)">{{n}}</a>
				</div>

				<select class="browser-default" @change="onTableLength">
					<option v-for="(option, index) in perPageOptions"
						:key="index"
						:value="option"
						:selected="option == currentPerPage"
					>
						{{ option === -1 ? lang['all'] : option }} {{ lang['rows'] }}
					</option>
				</select>

			</div>
			<div :class="{'datatable-info': true, 'rtl': lang.__is_rtl}" v-if="showRecordsCount">
				<span>{{ (currentPage - 1) * currentPerPage ? (currentPage - 1) * currentPerPage : 1 }}
					-{{ Math.min(processedRows.length, currentPerPage * currentPage) }}
				</span>
				<span>
					{{ lang['out_of_pages'] }}
				</span>
				<span>
					{{ processedRows.length }}
				</span>
			</div>
			<div class="buttons" :class="{'disabled': processedRows.length <= currentPerPage * currentPage}">
				<a href="javascript:undefined" @click.prevent="nextPage"> > </a>
				<a href="javascript:undefined" @click.prevent="lastPage"> >> </a>
			</div>
		</div>

		<div style="display: none" ref="print">
			table.print {
				width: 100%;
				border-collapse: collapse;
			}
			table.print tr td, table.print tr th {
				padding: 15px;
				border: 1px solid #e1e1e1;
				text-align: left;
			}
			.print-header {
				display: flex !important;
			}
			.print-header img {
				max-width: 100%;
			}
			.print-header div {
				flex-grow: 1;
			}
		</div>
	</div>
</template>

<script>
	import 'tb-skeleton/dist/skeleton.css';

	import Fuse from 'fuse.js';
	import locales from './locales';
	import { TbSkeleton } from 'tb-skeleton';

	export default {
		components: {
			TbSkeleton,
		},

		props: {
			title: {
				type: String,
				required: true,
			},
			className: {
				type: String,
				required: false,
				default: 'table'
			},
			columns: {
				type: Array,
				required: true,
			},
			rows: {
				type: Array,
				required: true,
			},

			clickable: {
				type: Boolean,
				required: false,
				default: true,
			},

			customButtons: {
				type: Array,
				required: false,
				default: () => [],
			},

			perPage: {
				type: Array,
				required: false,
				default: () => [10, 20, 30, 40, 50],
			},

			defaultPerPage: {
				type: Number,
				required: false,
				default: null,
			},

			sortable: {
				type: Boolean,
				required: false,
				default: true,
			},

			searchable: {
				type: Boolean,
				required: false,
				default: true,
			},

			exactSearch: {
				type: Boolean,
				required: false,
				default: false,
			},

			serverSearch: {
				type: Boolean,
				required: false,
				default: false,
			},

			serverSearchFunc: {
				type: Function,
				required: false,
				default: () => {},
			},

			paginate: {
				type: Boolean,
				required: false,
				default: true,
			},

			exportable: {
				type: Boolean,
				required: false,
				default: true,
			},

			printable: {
				type: Boolean,
				required: false,
				default: true,
			},

			locale: {
				type: String,
				required: false,
				default: 'en',
			},

			initSortCol: {
				type: Number,
				required: false,
				default: -1,
			},

			loadingAnimation: {
				type: Boolean,
				required: false,
				default: true,
			},

			showRecordsCount: {
				type: Boolean,
				required: false,
				default: false,
			}

		},

		data: () => ({
			currentPage: 1,
			currentPerPage: 10,
			sortColumn: -1,
			sortType: 'asc',
			searching: false,
			searchInput: '',
			win: null
		}),

		methods: {
			nextPage() {
				if (this.processedRows.length > this.currentPerPage * this.currentPage)
					++this.currentPage;
			},

			previousPage() {
				if (this.currentPage > 1)
					--this.currentPage;
			},

			firstPage() {
				this.currentPage = 1;
			},

			lastPage() {
				this.currentPage = Math.ceil(this.processedRows.length / this.currentPerPage);
			},

			goToPage(n) {
				if (n >= 1 && n <= Math.ceil(this.processedRows.length / this.currentPerPage)) this.currentPage = n;
			},

			onTableLength(e) {
				this.currentPerPage = parseInt(e.target.value);
				this.currentPage = 1;
			},

			sort(index) {
				if (!this.sortable)
					return;
				if (this.sortColumn === index) {
					this.sortType = this.sortType === 'asc' ? 'desc' : 'asc';
				} else {
					this.sortType = 'asc';
					this.sortColumn = index;
				}
			},

			search(e) {
				this.searching = !this.searching;
			},

			click(row) {
				if(!this.clickable){
					return;
				}

				if(getSelection().toString()){
					// Return if some text is selected instead of firing the row-click event.
					return;
				}

				this.$emit('row-click', row);
			},

			generateCSV() {

			},
			exportExcel() {
				var csv = [];
				var doc = new DOMParser().parseFromString(this.renderTable(), "text/html");
				var rows = doc.querySelectorAll("table tr");

				for (var i = 0; i < rows.length; i++) {
					var row = [], cols = rows[i].querySelectorAll("td, th");

					for (var j = 0; j < cols.length; j++)
						row.push(cols[j].innerText);

					csv.push(row.join(","));
				}

				csv = csv.join("\n");

				 // CSV FILE
				var csvFile = new Blob([csv], {type: "text/csv"});

				// Download link
				var downloadLink = document.createElement("a");

				const documentPrefix = this.title != '' ? this.title.replace(/ /g, '-') : 'Sheet';
				const d = new Date();

				var filename = documentPrefix
					+ '-' + d.getFullYear() + '-' + (d.getMonth() + 1) + '-' + d.getDate()
					+ '-' + d.getHours() + '-' + d.getMinutes() + '-' + d.getSeconds()
					+ '.csv';

				// File name
				downloadLink.download = filename;

				// We have to create a link to the file
				downloadLink.href = window.URL.createObjectURL(csvFile);

				downloadLink.style.display = "none";
				document.body.appendChild(downloadLink);
				downloadLink.click();

				// const mimeType = 'data:application/vnd.ms-excel';
				// const html = this.renderTable().replace(/ /g, '%20');

				// // eslint-disable-next-line eqeqeq
				// const documentPrefix = this.title != '' ? this.title.replace(/ /g, '-') : 'Sheet';
				// const d = new Date();

				// const dummy = document.createElement('a');
				// dummy.href = mimeType + ', ' + html;
				// dummy.download = documentPrefix
				// 	+ '-' + d.getFullYear() + '-' + (d.getMonth() + 1) + '-' + d.getDate()
				// 	+ '-' + d.getHours() + '-' + d.getMinutes() + '-' + d.getSeconds()
				// 	+ '.xls';
				// document.body.appendChild(dummy);
				// dummy.click();

			},

			print(target) {
				const clonedTable = this.$refs.table.cloneNode(true);
				clonedTable.classList.add("print");
				//this.synchronizeCssStyles(this.$refs.table, clonedTable, true);

				clonedTable.style.maxWidth = '100%';
				clonedTable.style.boxShadow = '0px 0px 0px 1px rgba(0,0,0,0.2)';
				clonedTable.style.margin = '8px auto';


				let win = window.open('', target);
				this.win = win;

				const style = this.$refs.print.cloneNode(true);

				const header = this.$refs['print-header'].cloneNode(true);

				var html = '<style>'+style.innerHTML+'</style>' + header.outerHTML + clonedTable.outerHTML;

				win.document.body.style.fontFamily = '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen-Sans, Ubuntu, Cantarell, "Helvetica Neue", sans-serif';
				win.document.body.innerHTML = (html);

				win.document.body.load = win.print();
			},

			renderTable() {
				let table = '<table><thead>';

				table += '<tr>';
				for (let i = 0; i < this.columns.length; i++) {
					const column = this.columns[i];
					table += '<th>';
					table += 	column.label;
					table += '</th>';
				}
				table += '</tr>';

				table += '</thead><tbody>';

				for (let i = 0; i < this.rows.length; i++) {
					const row = this.rows[i];
					table += '<tr>';
					for (let j = 0; j < this.columns.length; j++) {
						const column = this.columns[j];
						table += '<td>';
						table +=	this.collect(row, column.field);
						table += '</td>';
					}
					table += '</tr>';
				}

				table += '</tbody></table>';

				return table;
			},

			dig(obj, selector) {
				let result = obj;
				const splitter = selector.split('.');

				for (let i = 0; i < splitter.length; i++) {
					if (result == undefined)
						return undefined;

					result = result[splitter[i]];
				}

				return result;
			},

			collect(obj, field) {
				if (typeof(field) === 'function')
					return field(obj);
				else if (typeof(field) === 'string')
					return this.dig(obj, field);
				else
					return undefined;
			},
		},

		watch: {
			perPageOptions(newOptions, oldOptions) {
				// If defaultPerPage is provided and it's a valid option, set as current per page
				if (newOptions.indexOf(this.defaultPerPage) > -1) {
					this.currentPerPage = parseInt(this.defaultPerPage);
				} else {
					// Set current page to first value
					this.currentPerPage = newOptions[0];
				}
			},

			searchInput(newSearchInput) {
				if (this.searching && this.serverSearch && this.serverSearchFunc)
					this.serverSearchFunc(newSearchInput);
			},
		},

		computed: {
			perPageOptions() {
				let options = (Array.isArray(this.perPage) && this.perPage) || [10, 20, 30, 40, 50];

				// Force numbers
				options = options.map(v => parseInt(v));


				// Sort options
				options.sort((a,b) => a - b);

				// And add "All"
				options.push(-1);


				return options;
			},

			processedRows() {
				let computedRows = this.rows;

				if (this.sortable !== false)
					computedRows = computedRows.sort((x,y) => {
						if (!this.columns[this.sortColumn])
							return 0;

						const cook = x => {
							x = this.collect(x, this.columns[this.sortColumn].field);
							if (typeof(x) === 'string') {
								x = x.toLowerCase();
								if (this.columns[this.sortColumn].numeric)
									x = x.indexOf('.') >= 0 ? parseFloat(x) : parseInt(x);
							}
							return x;
						};

						x = cook(x);
						y = cook(y);

						return (x < y ? -1 : (x > y ? 1 : 0)) * (this.sortType === 'desc' ? -1 : 1);
					});

				if (this.searching && !this.serverSearch && this.searchInput) {
					const searchConfig = { keys: this.columns.map(c => c.field) };

					// Enable searching of numbers (non-string)
					// Temporary fix of https://github.com/krisk/Fuse/issues/144
					searchConfig.getFn = (obj, path) => {
						const property = this.dig(obj, path);
						if(Number.isInteger(property))
							return JSON.stringify(property);
						return property;
					};

					if (this.exactSearch) {
						//return only exact matches
						searchConfig.threshold = 0,
						searchConfig.distance = 0;
					}

					computedRows = (new Fuse(computedRows, searchConfig)).search(this.searchInput);
				}

				return computedRows;
			},

			paginated() {
				let paginatedRows = this.processedRows;

				if (this.paginate && this.currentPerPage !== -1)
					paginatedRows = paginatedRows.slice(
						(this.currentPage - 1) * this.currentPerPage,
						this.currentPerPage === -1 ? paginatedRows.length + 1 : this.currentPage * this.currentPerPage
					);

				return paginatedRows;
			},

			lang() {
				return this.locale in locales ? locales[this.locale] : locales['en'];
			},
		},

		mounted() {
			if (!(this.locale in locales))
				console.error(`vue-materialize-datable: Invalid locale '${this.locale}'`);
			this.sortColumn = this.initSortCol;
		},
	};
</script>

<style scoped lang="scss" rel="stylesheet/scss">
	div.datatable-wrapper {
		padding: 0;
	}

	tr.clickable {
		cursor: pointer;
	}

	#search-input {
		margin: 0;
		border: transparent 0 !important;
		height: 48px;
		color: rgba(0, 0, 0, .84);
	}

	#search-input-container {
		padding: 0 14px 0 24px;
		border-bottom: solid 1px #DDDDDD;
	}

	table {
		table-layout: fixed;
		margin-bottom: 0;
	}

	.table-header {
		align-items: center;
		display: flex;
		border-bottom: solid 1px #DDDDDD;
	}

	.table-header .actions {
		display: -webkit-flex;
		margin-left: auto;
	}

	.table-header .btn-flat {
		min-width: 36px;
		padding: 0 8px;
	}

	.table-header input {
		margin: 0;
		height: auto;
	}

	.table-header i {
		color: rgba(0, 0, 0, 0.54);
		font-size: 24px;
	}

	.table-footer {
		display: flex;
		flex-direction: row;
		justify-content: space-between;
		align-items: center;
		color: rgba(0, 0, 0, 0.54);
		//border-top: 1px solid #E1E1E1;
		box-shadow: 0px 0px 5px 0px rgba(0,0,0,0.1);
		> div {
			display: flex;
			align-items: center;
		}
	}

	.table-footer .datatable-length {
		display: flex;
		select {
			outline: none;
			border: 1px solid #e1e1e1;
			padding: 3px;
		}
		.datatable-page-link {
			display: flex;
			a {
				border: 1px solid #e1e1e1;
				font-size: 12px;
				padding: 2px;
				min-width: 20px;
				margin: 0 1px;
				text-align: center;
				display: block;
				text-decoration: none;
				&.selected {
					background-color: #e1e1e1;
				}
			}
		}

	}
	.buttons {
		padding: 3px;
		font-size: 11px;
		cursor: default;
		a {
			margin: 1px;
			display: inline-block;
			background-color: #E1E1E1;
			padding: 7.5px 5px;
			color: #525252;
			text-decoration: none;
		}
		&.disabled {
			a {
				background-color: #EFEFEF;
				color: darken(#EFEFEF, 20);
				cursor: default;
			}
		}
	}

	.table-footer label {
		font-size: 12px;
		color: rgba(0, 0, 0, 0.54);
		display: -webkit-flex;
		display: flex;
		-webkit-flex-direction: row;
		/* works with row or column */

		flex-direction: row;
		-webkit-align-items: center;
		align-items: center;
		-webkit-justify-content: center;
		justify-content: center;
	}

	.table-footer .select-wrapper {

	}

	.table-footer .datatable-info,
	.table-footer .datatable-length {

	}

	.table-footer .datatable-pagination {
		display: flex;
		-webkit-display: flex;
		margin: 0;
	}

	.table-footer .datatable-pagination li a {
		color: rgba(0, 0, 0, 0.54);
		padding: 0 8px;
		font-size: 24px;
	}

	.table-footer .select-wrapper input.select-dropdown {
		margin: 0;
		border-bottom: none;
		height: auto;
		line-height: normal;
		font-size: 12px;
		width: 40px;
		text-align: right;
	}

	.table-footer select {
		background-color: transparent;
		width: auto;
		padding: 0;
		border: 0;
		border-radius: 0;
		height: auto;
		margin-left: 20px;
	}

	.table-title {
		font-size: 20px;
		color: #000;
	}

	.rtl {
		direction: rtl;
	}

	.datatable-wrapper {
		border: none;
		.table-header {
			border-bottom: none ;
			height: auto ;
			.actions {
				height: auto;
			}
		}
		table {
			border: 1px solid #dee2e6;
			width: 100%;
			thead {
				th {
					border: 1px solid #dee2e6;
					border-bottom: none;
					position: relative;
					padding: 0.75rem !important;
					font-weight: bold;
					font-family: "Roboto Condensed";
					font-size: 1rem !important;
					color: #212529;
					overflow: visible;

					&.sorting {
						cursor: pointer;
					}
					&:after {
						display: none !important;
					}
					&.sorting-asc {
						&:before {
							content: "";
							height: 3px;
							background: #5B5B5B;
							display: block;
							position: absolute;
							top: -1px;
							left: -1px;
							right: -1px;
							width: calc(100% + 2px);
						}
					}
					&.sorting-desc {
						&:before {
							content: "";
							height: 4px;
							background: #5B5B5B;
							display: block;
							position: absolute;
							bottom: 0px;
							left: -1px;
							right: -1px;
							width: calc(100% + 2px);
						}
					}

				}
			}
			tbody {
				td {
					padding: 0.75rem !important;
					border: none !important;
					font-size: 1rem !important;

				}
				tr:nth-of-type(even) {
					background-color: rgba(0, 0, 0, 0.05);
				}
			}
		}

	}
	.table-header {
		border: none;
	}
	table .print {
		tr {
			td, th {
				border: 1px solid #e1e1e1 !important;
			}
		}
	}
</style>
