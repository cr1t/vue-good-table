<template>
  <div>
    <table ref="table" :class="styleClass">
      <thead>
        <tr v-if="globalSearch && externalSearchQuery == null">
          <td :colspan="lineNumbers ? columns.length + 1: columns.length">
            <div class="global-search">
              <span class="global-search-icon">
                <img src="../images/search_icon.png" alt="Search Icon" />
              </span>
              <input type="text" class="form-control global-search-input" :placeholder="globalSearchPlaceholder" v-model="globalSearchTerm" @keyup.enter="searchTable()" />
            </div>
          </td>
        </tr>
        <tr class="sortable">
          <th v-if="showSelection" class="line-numbers"></th>
          <th v-if="lineNumbers" class="line-numbers"></th>
          <th v-for="(column, index) in columns"
            :key="column.field" 
              @click="sort(index)"
              :class="columnHeaderClass(column, index)"
              :style="{width: column.width ? column.width : 'auto'}"
              v-if="!column.hidden">
            <slot name="table-column" :column="column">
              <span>{{column.label}}</span>
            </slot>
          </th>
          <slot name="thead-tr"></slot>
        </tr>
        <tr v-if="hasFilterRow">
          <th v-if="showSelection" class="line-numbers">
            
          </th>
          <th v-if="lineNumbers"></th>
          <th v-for="(column, index) in columns" :key="column.field"  v-if="!column.hidden">
            <div v-if="column.filterable">
              <input v-if="!column.filterDropdown"
                     type="text" class="form-control" :placeholder="'Filter ' + column.label"
                     :value="columnFilters[column.field]"
                     v-on:input="updateFilters(column, $event.target.value)">
              <select v-if="column.filterDropdown" class="form-control"
                      :value="columnFilters[column.field]"
                      v-on:input="updateFilters(column, $event.target.value)">
                <option value=""></option>
                <option v-for="option in column.filterOptions"  :key="option.value" :value="option.value">{{ option.text }}</option>
              </select>
            </div>
          </th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(row, index) in paginated"  :key="index" :class="onClick || showSelection ? 'clickable' : ''" @click="click(row, index)">
          <th v-if="showSelection" class="row-controls"><label class="custom-radio nowrap">
            <input type="radio" name="tableRadioSelection" v-bind:value="index" v-bind:id="index" v-model="selectedIndex"  >
            <span></span></label></th>
          <th v-if="lineNumbers" class="row-controls" >{{ getCurrentIndex(index) }}</th>
          <slot name="table-row-before" :row="row" :index="index"></slot>
          <slot name="table-row" :row="row" :formattedRow="formattedRow(row)" :index="index">
            <td v-for="(column, i) in columns"  :key="column.field"  :class="getDataStyle(i, 'td')"  v-if="!column.hidden">
              <span v-if="!column.html">{{ collectFormatted(row, column) }}</span>
              <span v-if="column.html" v-html="collect(row, column.field)"></span>
            </td>
          </slot>
          <slot name="table-row-after" :row="row" :index="index"></slot>
        </tr>
        <tr v-if="processedRows.length === 0">
          <td :colspan="columns.length">
            <slot name="emptystate">
              <div class="center-align text-disabled">
                No data for table.
              </div>
            </slot>
          </td>
        </tr>
      </tbody>
    </table>
    <br/>
    <div class="row">
      <div class="col-sm-2">
        <span>{{rowsPerPageText}} </span>
        <span v-if="perPage" class="perpage-count">{{perPage}} </span>
        <label>
            <a v-if="!perPage" class="clickCursor" :class="{selected:currentPerPage==10}" v-on:click="onTableLength(10)">10</a>&nbsp;
            <a v-if="!perPage" class="clickCursor" :class="{selected:currentPerPage==20}" v-on:click="onTableLength(20)">20</a>&nbsp;
            <a v-if="!perPage" class="clickCursor" :class="{selected:currentPerPage==30}" v-on:click="onTableLength(30)">30</a>&nbsp;
            <a v-if="!perPage" class="clickCursor" :class="{selected:currentPerPage==40}" v-on:click="onTableLength(40)">40</a>&nbsp;
            <a v-if="!perPage" class="clickCursor" :class="{selected:currentPerPage==50}" v-on:click="onTableLength(50)">50</a>&nbsp;
            <a v-if="!perPage" class="clickCursor" :class="{selected:currentPerPage==-1}" v-on:click="onTableLength(-1)">{{allText}}</a>
        </label>
      </div>
      <div class="col-sm-10">
        <div class="pull-right" style="padding-top:5px;" >
          <a href="javascript:undefined" class="page-btn" @click.prevent.stop="previousPage" tabindex="0">
            <span class="fa fa-chevron-circle-left " v-bind:class="{ 'left': !rtl, 'right': rtl }"></span>
            <span> {{prevText}}</span>
          </a>
          <span class="info">{{paginatedInfo}}</span>
          <a href="javascript:undefined" class="page-btn" @click.prevent.stop="nextPage" tabindex="0">
            <span>{{nextText}} </span>
            <span class="fa fa-chevron-circle-right" v-bind:class="{ 'right': !rtl, 'left': rtl }"></span>
          </a>
        </div>
      </div>

    </div>
  </div>
</template>
<script>
  import { format, parse, compareAsc } from 'date-fns/esm'
  export default {
    name: 'vue-good-table',
    props: {
      styleClass: { default: 'table ' },
      title: '',
      columns: {},
      rows: {},
      onClick: {},
      perPage: {},
      showSelection: { default: true },
      sortable: { default: true },
      paginate: { default: false },
      lineNumbers: { default: false },
      defaultSortBy: { default: null },
      responsive: { default: true },
      rtl: { default: false },

      // search
      globalSearch: { default: false },
      searchTrigger: { default: null },
      externalSearchQuery: { default: null },

      // text options
      globalSearchPlaceholder: { default: 'Search Table' },
      nextText: { default: 'Next' },
      prevText: { default: 'Prev' },
      rowsPerPageText: { default: 'Rows per page :' },
      ofText: { default: 'of' },
      allText: { default: 'All' },

    },

    data: () => ({
      currentPage: 1,
      currentPerPage: 10,
      sortColumn: -1,
      sortType: 'asc',
      globalSearchTerm: '',
      columnFilters: {},
      filteredRows: [],
      timer: null,
      forceSearch: false,
      sortChanged: false,
      selectedIndex:-1
    }),

    methods: {
      nextPage() {
        if (this.currentPerPage == -1) return;
        if (this.processedRows.length > this.currentPerPage * this.currentPage)
          ++this.currentPage;
        this.pageChanged();
        this.clearSelection()
      },

      previousPage() {
        if (this.currentPage > 1)
          --this.currentPage;
        this.pageChanged();
        this.clearSelection()
      },

      pageChanged() {
        this.$emit('pageChanged', { currentPage: this.currentPage, total: Math.floor(this.rows.length / this.currentPerPage) });
      },

      onTableLength(value) {
        this.currentPerPage = value;
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
        this.sortChanged = true;
        this.clearSelection()
      },

      click(row, index) {
        if (this.onClick)
          this.onClick(row, index);

        if (this.showSelection) {
          this.selectedIndex = index;
        }
      },

      clearSelection() {
        this.selectedIndex = -1;
        if (this.onClick)
          this.onClick({}, -1);

        if (this.showSelection) {
          this.onClick({}, -1);
        }
      },

      searchTable() {
        if (this.searchTrigger == 'enter') {
          this.forceSearch = true;
          this.sortChanged = true;
          this.clearSelection()
        }
      },

      // field can be:
      // 1. function
      // 2. regular property - ex: 'prop'
      // 3. nested property path - ex: 'nested.prop'
      collect(obj, field) {

        //utility function to get nested property
        function dig(obj, selector) {
          var result = obj;
          const splitter = selector.split('.');
          for (let i = 0; i < splitter.length; i++)
            if (typeof (result) === 'undefined')
              return undefined;
            else
              result = result[splitter[i]];
          return result;
        }

        if (typeof (field) === 'function')
          return field(obj);
        else if (typeof (field) === 'string')
          return dig(obj, field);
        else
          return undefined;
      },

      collectFormatted(obj, column) {
        //helper functions within collect
        function formatDecimal(v) {
          return parseFloat(Math.round(v * 100) / 100).toFixed(2);
        }

        function formatPercent(v) {
          return parseFloat(v * 100).toFixed(2) + '%';
        }

        function formatDate(v) {
          // convert to date
          if (v == null)
            return "";
          else
            return moment(v).format('YYYY.MM.DD');
        }

        function formatBool(v)
        {
          if(v == true)
            return "Yes"
          else
            return "No";
        }
        var value = this.collect(obj, column.field);

        if (value === undefined) return '';
        //lets format the resultant data
        switch (column.type) {
          case 'decimal':
            return formatDecimal(value);
          case 'percentage':
            return formatPercent(value);
          case 'date':
            return formatDate(value);
          case 'boolean':
            return formatBool(value);
          default:
            return value;
        }
      },

      formattedRow(row) {
        var formattedRow = {};
        for (const col of this.columns) {
          if (col.field) {
            formattedRow[col.field] = this.collectFormatted(row, col);
          }
        }
        return formattedRow;
      },

      // Get the necessary style-classes for the given column
      //--------------------------------------------------------
      columnHeaderClass(column, index) {
        var classString = '';
        if (this.sortable) {
          classString += 'sorting ';
        }
        if (index === this.sortColumn) {
          if (this.sortType === 'desc') {
            classString += 'st-sort-descent ';
          } else {
            classString += 'st-sort-ascent ';
          }
        }
        classString += this.getDataStyle(index, 'th');
        return classString;
      },
      // given column index, we can figure out what style classes
      // to apply to this data
      //---------------------------------------------------------
      getDataStyle(index, type) {
        var classString = '';
        switch (this.columns[index].type) {
          case 'number':
          case 'percentage':
          case 'decimal':
          case 'date':
          case 'text':
            classString += 'right-align ';
            break;
          default:
            classString += 'left-align ';
            break;
        }
        if (typeof type !== 'undefined' && this.columns[index].hasOwnProperty(type + 'Class')) {
          classString += ' ';
          classString = this.columns[index][type + 'Class'];
        }
        return classString;
      },

      //since vue doesn't detect property addition and deletion, we
      // need to create helper function to set property etc
      updateFilters(column, value) {
        const _this = this;
        if (this.timer) clearTimeout(this.timer);
        this.timer = setTimeout(function () {
          _this.$set(_this.columnFilters, column.field, value)
        }, 400);
      },

      //method to filter rows
      filterRows() {
        var computedRows = JSON.parse(JSON.stringify(this.rows));
        // we need to preserve the original index of rows so lets do that
        for (const [index, row] of computedRows.entries()) {
          row.originalIndex = index;
        }

        if (this.hasFilterRow) {
          for (var col of this.columns) {
            if (col.filterable && this.columnFilters[col.field]) {
              computedRows = computedRows.filter(row => {

                // If column has a custom filter, use that.

                if (col.filter) {
                  return col.filter(this.collect(row, col.field), this.columnFilters[col.field])
                }

                // Use default filters

                switch (col.type) {
                  case 'number':
                  case 'percentage':
                  case 'decimal':
                    //in case of numeric value we need to do an exact
                    //match for now`
                    return this.collect(row, col.field) == this.columnFilters[col.field];  
                  case 'boolean':
                    return this.collect(row, col.field) == eval(this.columnFilters[col.field]);  
                  default:
                    //text value lets test starts with
                    return this.collect(row, col.field)
                      .toLowerCase()
                      .startsWith(
                      (this.columnFilters[col.field]).toLowerCase()
                      );
                }
              });
            }
          }
        }
        this.filteredRows = computedRows;

        this.clearSelection()
      },

      getCurrentIndex(index) {
        return (this.currentPage - 1) * this.currentPerPage + index + 1;
      },
    },

    watch: {
      columnFilters: {
        handler: function (newObj) {
          this.filterRows();
        },
        deep: true,
      },
      rows: {
        handler: function (newObj) {
          this.filterRows();
        },
        deep: true,
      },
      perPage() {
        if (this.perPage) {
          this.currentPerPage = this.perPage;
        } else {
          //reset to default
          this.currentPerPage = 10;
        }
      }
    },

    computed: {
      searchTerm() {
        return (this.externalSearchQuery != null) ? this.externalSearchQuery : this.globalSearchTerm;
      },

      //
      globalSearchAllowed() {
        if (this.globalSearch
          && !!this.globalSearchTerm
          && this.searchTrigger != 'enter') {
          return true;
        }

        if (this.externalSearchQuery != null
          && this.searchTrigger != 'enter') {
          return true;
        }

        if (this.forceSearch) {
          this.forceSearch = false;
          return true;
        }

        return false;
      },

      // to create a filter row, we need to
      // make sure that there is atleast 1 column
      // that requires filtering
      hasFilterRow() {
        if (!this.globalSearch) {
          for (var col of this.columns) {
            if (col.filterable) {
              return true;
            }
          }
        }
        return false;
      },

      // this is done everytime sortColumn
      // or sort type changes
      //----------------------------------------
      processedRows() {
        var computedRows = this.filteredRows;

        // take care of the global filter here also
        if (this.globalSearchAllowed) {
          var filteredRows = [];
          for (var row of this.rows) {
            for (var col of this.columns) {
              if (String(this.collectFormatted(row, col)).toLowerCase()
                .search(this.searchTerm.toLowerCase()) > -1) {
                filteredRows.push(row);
                break;
              }
            }
          }
          computedRows = filteredRows;
        }

        //taking care of sort here only if sort has changed
        if (this.sortable !== false && this.sortColumn !== -1 &&

          // if search trigger is enter then we only sort
          // when enter is hit
          (this.searchTrigger != 'enter' || this.sortChanged)) {

          this.sortChanged = false;

          computedRows = computedRows.sort((x, y) => {
            if (!this.columns[this.sortColumn])
              return 0;

            const cook = (d) => {
              d = this.collect(d, this.columns[this.sortColumn].field);

              //take care of dates too.
              if (this.columns[this.sortColumn].type === 'date') {
                d = parse(d + '', this.columns[this.sortColumn].inputFormat, new Date());
              } else if (typeof (d) === 'string') {
                d = d.toLowerCase();
                if (this.columns[this.sortColumn].type == 'number')
                  d = d.indexOf('.') >= 0 ? parseFloat(d) : parseInt(d);
              }
              return d;
            }

            x = cook(x);
            y = cook(y);

            // date comparison here
            if (this.columns[this.sortColumn].type === 'date') {
              return (compareAsc(x, y)) * (this.sortType === 'desc' ? -1 : 1);
            }

            // regular comparison here
            return (x < y ? -1 : (x > y ? 1 : 0)) * (this.sortType === 'desc' ? -1 : 1);
          })
        }

        // if the filtering is event based, we need to maintain filter
        // rows
        if (this.searchTrigger == 'enter') {
          this.filteredRows = computedRows;
        }

        return computedRows;
      },

      paginated() {
        var paginatedRows = this.processedRows;

        if (this.paginate) {
          var pageStart = (this.currentPage - 1) * this.currentPerPage;

          // in case of filtering we might be on a page that is
          // not relevant anymore
          // also, if setting to all, current page will not be valid
          if (pageStart >= this.processedRows.length
            || this.currentPerPage == -1) {
            this.currentPage = 1;
            pageStart = 0;
          }

          //calculate page end now
          var pageEnd = paginatedRows.length + 1;

          //if the setting is set to 'all'
          if (this.currentPerPage != -1) {
            pageEnd = this.currentPage * this.currentPerPage;
          }

          paginatedRows = paginatedRows.slice(pageStart, pageEnd);
        }
        return paginatedRows;
      },

      paginatedInfo() {
        var infoStr = '';
        infoStr += (this.currentPage - 1) * this.currentPerPage ? (this.currentPage - 1) * this.currentPerPage : 1;
        infoStr += ' - ';
        infoStr += Math.min(this.processedRows.length, this.currentPerPage * this.currentPage);
        infoStr += ' ' + this.ofText + ' ';
        infoStr += this.processedRows.length;
        if (this.currentPerPage == -1) {
          return '1 - ' + this.processedRows.length + ' ' + this.ofText + ' ' + this.processedRows.length;
        }
        return infoStr;
      },
    },

    mounted() {
      this.filteredRows = JSON.parse(JSON.stringify(this.rows));

      // we need to preserve the original index of rows so lets do that
      for (const [index, row] of this.filteredRows.entries()) {
        row.originalIndex = index;
      }

      if (this.perPage) {
        this.currentPerPage = this.perPage;
      }

      //take care of default sort on mount
      if (this.defaultSortBy) {
        for (let [index, col] of this.columns.entries()) {
          if (col.field === this.defaultSortBy.field) {
            this.sortColumn = index;
            this.sortType = this.defaultSortBy.type || 'asc';
            this.sortChanged = true;
            break;
          }
        }
      }
    }
  }
</script>
