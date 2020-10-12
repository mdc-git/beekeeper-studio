<template>
  <div class="result-table">
    <div ref="tabulator"></div>
  </div>
</template>

<script type="text/javascript">
  import Tabulator from 'tabulator-tables'
  import _ from 'lodash'
  import dateFormat from 'dateformat'
  import Converter from '../../mixins/data_converter'
  import Mutators from '../../mixins/data_mutators'

  export default {
    mixins: [Converter, Mutators],
    data() {
      return {
        tabulator: null
      }
    },
    props: ['result', 'tableHeight', 'query', 'active'],
    watch: {
      active() {
        if (!this.tabulator) return;
        if (this.active) {
          this.tabulator.restoreRedraw()
          this.$nextTick(() => {
            this.tabulator.redraw()
          })
        } else {
          this.tabulator.blockRedraw()
        }
      },
      tableData: {
        handler() {
          // this.tabulator.replaceData(this.tableData)
        }
      },
      tableColumns: {
        handler() {
          this.tabulator.setColumns(this.tableColumns)
        }
      },
      tableHeight() {
        this.tabulator.setHeight(this.actualTableHeight)
      }
    },
    computed: {
      tableData() {
          return this.dataToTableData(this.result, this.tableColumns)
      },
      tableTruncated() {
          return this.result.truncated
      },
      tableColumns() {
        const columnWidth = this.result.fields.length > 20 ? 125 : undefined
        return this.result.fields.map((column) => {
          const result = {
            title: column.name,
            field: column.name,
            dataType: column.dataType,
            width: columnWidth,
            mutatorData: this.resolveDataMutator(column.dataType),
            formatter: this.cellFormatter
          }
          return result;
        })
      },
      actualTableHeight() {
        return '100%'
        // let result = this.tableHeight
        // if (this.tableHeight == 0) {
        //   result = '100%'
        // }
        // return result
      },
    },
    beforeDestroy() {
      if (this.tabulator) {
        this.tabulator.destroy()
      }
    },
    async mounted() {
      this.tabulator = new Tabulator(this.$refs.tabulator, {
        ajaxURL: "http://fake",
        ajaxSorting: true,
        ajaxFiltering: true,
        pagination: "remote",
        paginationSize: this.limit,
        ajaxRequestFunc: this.dataFetch,

        reactiveData: false,
        virtualDomHoz: true,
        columns: this.tableColumns, //define table columns
        height: this.actualTableHeight,
        nestedFieldSeparator: false,
        cellClick: this.cellClick,
        clipboard: true,
        keybindings: {
          copyToClipboard: false
        },
        downloadConfig: {
          columnHeaders: true
        }
      });
    },
    methods: {
      dataFetch(url,config,params) {
      
      const result = new Promise((resolve, reject) => {
        (async () => {
          try {
            const page = params?.page ?? 1
            const offset = (page-1) * 1000
            const end = Math.min(offset + 1000,this.result.rows.length)
          
            let rows = [];

            if (params.sorters[0]) {
              const sortcolumn = params?.sorters[0]?.field
              if (!this.sortcolumn || sortcolumn !== this.sortcolumn) {
                this.result.rows = _.sortBy( this.result.rows, sortcolumn )
                this.sortcolumn = sortcolumn
              }
              if (this.page === page || !this.page) {
                this.togglesort = this.togglesort === "asc" ? "desc" : "asc";
                this.result.rows.reverse()
              }
            }
            
            this.page = page;

            for (let i = offset; i < end; i++) {
              rows.push(this.result.rows[i])
            }

            const data = this.dataToTableData({rows:rows}, this.tableColumns);
            resolve({
              last_page: Math.ceil(this.result.rows.length/1000),
              data,
            })
          } catch (error) {
            reject();
            throw error;
          }
        })();
      });
      return result;
    },
      cellClick(e, cell) {
        this.selectChildren(cell.getElement())
      },
      download(format) {
        const dateString = dateFormat(new Date(), 'yyyy-mm-dd_hMMss')
        const title = this.query.title ? _.snakeCase(this.query.title) : "query_results"
        var fs = require("fs")
            const writer = fs.createWriteStream(`/home/sudoer/Downloads/exports/${title}-${dateString}.${format}`);
        
        for (let i =0; i< this.result.rows.length; i++) {
          const row = this.result.rows[i]
          var item = [];
          Object.values(row).forEach((col) => {
            if(col){
              switch(typeof col){
                case "object":
                col.value = JSON.stringify(col);
                break;

                case "undefined":
                case "null":
                col = "";
                break;
              }

              item.push('"' + String(col).split('"').join('""') + '"');
            }
            
          });

          let values = item.join(',')
          values = values + '\n';
          //console.log(values)
          writer.write(values)
        }
        
         writer.close()
      },
      clipboard() {
        this.tabulator.copyToClipboard("table", true)
        this.$noty.info("Table data copied to clipboard")
      }
    }
	}
</script>
