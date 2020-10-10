<template>
  <div class="result-table">
    <div ref="tabulator"></div>
    <statusbar class="tabulator-footer">
      <div class="col x12 flex flex-center">
        <span
          ref="paginationArea"
          class="tabulator-paginator"
          v-show="this.meta.count > this.meta.limit"
        ></span>
      </div>
    </statusbar>
  </div>
</template>

<script type="text/javascript">
import Tabulator from "tabulator-tables";
import Converter from "../../mixins/data_converter";
import Mutators from "../../mixins/data_mutators";
import Statusbar from "../common/StatusBar";
import _ from "lodash";
import dateFormat from "dateformat";
import Papa from "papaparse";
import FileSaver from "file-saver";

export default {
  components: { Statusbar },
  mixins: [Converter, Mutators],
  data() {
    return {
      tabulator: null,
      limit: 1000,
      totalRecords: 0,
      resultRecords: 0,
      togglesort: "asc",
    };
  },
  props: ["result", "tableHeight", "query", "active", "connection", "meta"],
  watch: {
    active() {
      if (!this.tabulator) return;
      if (this.active) {
        this.tabulator.restoreRedraw();
        this.$nextTick(() => {
          this.tabulator.redraw();
        });
      } else {
        this.tabulator.blockRedraw();
      }
    },
    tableData: {
      handler() {
        // this.tabulator.replaceData(this.tableData)
      },
    },
    tableColumns: {
      handler() {
        this.tabulator.setColumns(this.tableColumns);
      },
    },
    tableHeight() {
      this.tabulator.setHeight(this.actualTableHeight);
    },
  },
  computed: {
    statusbarMode() {
      return null;
    },

    tableTruncated() {
      return this.result.truncated;
    },

    actualTableHeight() {
      return "100%";
      // let result = this.tableHeight
      // if (this.tableHeight == 0) {
      //   result = '100%'
      // }
      // return result
    },
  },
  beforeDestroy() {
    if (this.tabulator) {
      this.tabulator.destroy();
    }
  },
  async mounted() {
    this.tabulator = new Tabulator(this.$refs.tabulator, {
      //data: this.tableData, //link data to table
      ajaxURL: "http://fake",
      ajaxSorting: true,
      ajaxFiltering: true,
      pagination: "remote",
      paginationSize: this.meta.limit,
      paginationElement: this.$refs.paginationArea,
      // callbacks
      ajaxRequestFunc: this.dataFetch,
      reactiveData: false,
      virtualDomHoz: true,
      //columns: this.tableColumns, //define table columns
      height: this.actualTableHeight,
      nestedFieldSeparator: false,
      cellClick: this.cellClick,
      clipboard: true,
      keybindings: {
        copyToClipboard: false,
      },
      downloadConfig: {
        columnHeaders: true,
      },
    });
  },
  methods: {
    dataFetch(url, config, params) {
      let limit, offset, orderBy, orderBy2;

      if (params.sorters[0]) {
        orderBy2 =
          " ORDER BY " +
          params.sorters[0].field +
          " " +
          (this.togglesort = this.togglesort === "asc" ? "desc" : "asc");
      } else {
        orderBy2 = "";
      }

      if (this.meta.orderby) {
        orderBy = " ORDER BY " + this.meta.orderby;
      } else {
        orderBy = "";
      }

      if (params.size) {
        limit = this.meta.limit || params.size;
      }

      if (params.page) {
        offset = this.meta.offset || (params.page - 1) * limit;
      }

      const result = new Promise((resolve, reject) => {
        (async () => {
          try {
            let sql =
              this.meta.basesql +
              orderBy +
              " LIMIT " +
              limit +
              " OFFSET " +
              offset;
            if (orderBy2 !== "") {
              sql = "SELECT * FROM ( " + sql + " ) res " + orderBy2;
            }
            console.log("->>>>", sql);
            const query = this.connection.query(sql);
            const response = await query.execute();
            Object.freeze(response);

            const count = this.meta.count;
            this.totalRecords = count;
            this.resultRecords = response[0].rows.length;
            const fields = response[0].fields;
            const columnWidth = fields.length > 20 ? 125 : undefined;
            const columns = fields.map((column) => {
              const result = {
                title: column.name,
                field: column.name,
                width: columnWidth,
                mutatorData: this.resolveDataMutator(
                  response[0].rows[0][column.name]
                ),
                formatter: this.cellFormatter,
              };
              return result;
            });
            this.tabulator.setColumns(columns);
            const data = this.dataToTableData(response[0], columns);
            Object.freeze(data);
            resolve({
              last_page: Math.ceil(this.meta.count / this.meta.limit),
              data,
            });
          } catch (error) {
            reject();
            throw error;
          }
        })();
      });
      return result;
    },
    cellClick(e, cell) {
      this.selectChildren(cell.getElement());
    },
    async download() {
      this.tabulator.modules.ajax.showLoader();
      this.tabulator.blockRedraw();

      const query = this.meta.basesql;
      const response = await this.connection.query(query).execute();
      Object.freeze(response);

      const dataString = Papa.unparse(response[0].rows);
      Object.freeze(dataString);
      const dateString = dateFormat(new Date(), "yyyy-mm-dd_hMMss");
      const title = this.query.title
        ? _.snakeCase(this.query.title)
        : "query_results";
      FileSaver.saveAs(
        new Blob([dataString], { type: "text/csv;charset=utf-8" }),
        `${title}-${dateString}.csv`
      );

      this.tabulator.modules.ajax.hideLoader();
      this.tabulator.restoreRedraw();
    },
    clipboard() {
      this.tabulator.copyToClipboard("table", true);
      this.$noty.info("Table data copied to clipboard");
    },
  },
};
</script>
