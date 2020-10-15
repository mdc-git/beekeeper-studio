<template>
  <div class="result-table">
    <div ref="tabulator"></div>
    <statusbar class="tabulator-footer">
      <div class="col x12 flex flex-center">
        <span
          ref="paginationArea"
          class="tabulator-paginator"
          v-show="this.meta.count > this.limit"
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
//import Papa from "papaparse";
//import FileSaver from "file-saver";

export default {
  components: { Statusbar },
  mixins: [Converter, Mutators],
  data() {
    return {
      tabulator: null,
      limit: 1000,
      togglesort: "asc",
      orderBy: "",
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
        //this.tabulator.setColumns(this.tableColumns);
      },
    },
    tableHeight() {
      this.tabulator.setHeight(this.actualTableHeight);
    },
  },
  computed: {
    actualTableHeight() {
      return "100%";
    },
  },
  beforeDestroy() {
    if (this.tabulator) {
      this.tabulator.destroy();
    }
  },
  async mounted() {
    this.tabulator = new Tabulator(this.$refs.tabulator, {
      ajaxURL: "http://fake",
      ajaxSorting: true,
      ajaxFiltering: true,
      pagination: "remote",
      paginationSize: this.limit,
      paginationElement: this.$refs.paginationArea,
      ajaxRequestFunc: this.dataFetch,
      reactiveData: false,
      virtualDomHoz: true,
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
      // page size
      const records_per_page = this.limit;
      // total result count
      const records = this.meta.count;
      // last page
      const last_page = Math.ceil(records / records_per_page);
      // current page
      const page = params?.page ?? 1;

      let limit, offset;

      let orderBy2;

      // column sorting
      if (params.sorters[0]) {
        if (this.page === page || !this.page) {
          this.togglesort = this.togglesort === "asc" ? "desc" : "asc";
        }
        this.page = page;
        orderBy2 = ` ORDER BY ${params.sorters[0].field} ${this.togglesort} `;
      } else {
        orderBy2 = "";
      }

      // limit, offset
      if (this.togglesort === "asc") {
        offset = (page - 1) * records_per_page;
        limit = page === last_page ? records - offset : records_per_page;
      } else {
        offset = page === last_page ? 0 : records - page * records_per_page;
        limit =
          page === last_page
            ? records - (page - 1) * records_per_page
            : records_per_page;
      }

      const result = new Promise((resolve, reject) => {
        (async () => {
          try {
            if (orderBy2 !== "") {
              limit = this.limit;
              offset = offset = (page - 1) * records_per_page;
            }
            let sql = `SELECT * FROM ( SELECT * FROM ( ${this.query.text} ) beekeeper_sort ${orderBy2} ) beekeper_limit LIMIT ${limit} OFFSET ${offset}`;
            console.log("->>>>", sql);
            const query = this.connection.query(sql);
            const response = await query.execute();
            Object.freeze(response);
            const fields = response[0].fields;
            const columnWidth = 125;
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
            const data = await this.dataToTableData(response[0], columns);
            resolve({
              last_page: last_page,
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
    async download(format) {
      const { app, dialog } = require("electron").remote;
      const dateString = dateFormat(new Date(), "yyyy-mm-dd_hMMss");
      const title = this.query.title
        ? _.snakeCase(this.query.title)
        : "query_results";
      const options = {
        title: "Save file",
        defaultPath:
          app.getPath("downloads") + `/${title}-${dateString}.${format}`,
        buttonLabel: "Save",

        filters: [
          { name: "csv", extensions: ["csv"] },
          { name: "All Files", extensions: ["*"] },
        ],
      };

      const file = await dialog.showSaveDialog(options);
      if(file.filePath==="") return
      var fs = require("fs");
      const writer = fs.createWriteStream(file.filePath);
      writer.on("error", function (err) {
        console.log(err);
      });
      var mysql = require("mysql2");
      const stream = require("stream");
      console.log(this.connection.server);
      var con = mysql.createConnection({
        host: this.connection.server.config.host,
        user: this.connection.server.config.user,
        password: this.connection.server.config.password,
        port: this.connection.server.config.port,
        database: this.connection.database.database,
      });

      con.connect(function (err) {
        if (err) throw err;
        console.log("Connected!");
      });
      const tranformRow = function (row) {
        var item = [];
        Object.values(row).forEach((col) => {
          if (col) {
            switch (typeof col) {
              case "object":
                col = JSON.stringify(col);
                break;
              case "undefined":
              case "null":
                col = "(NULL)";
                break;
            }
            item.push('"' + String(col).split('"').join('""') + '"');
          }
        });
        let values = item.join(",");
        values = values + "\n";
        return values;
      };
      let headerWritten = false;
      con
        .query(this.query.text)
        .stream()
        .pipe(
          new stream.Transform({
            objectMode: true,
            transform: function (row, encoding, callback) {
              // Do something with the row of data
              if (!headerWritten) {
                writer.write(tranformRow(Object.keys(row)));
                headerWritten = true;
              }

              writer.write(tranformRow(row));
              callback();
            },
          })
        )
        .on("finish", function () {
          con.end();
          writer.close();
        });

      const self = this;
      writer.on("finish", function () {
        self.$noty.info(
          "Download finished.<br>Saved file to:<br>" + file.filePath
        );
      });
    },
    clipboard() {
      this.tabulator.copyToClipboard("table", true);
      this.$noty.info("Table data copied to clipboard");
    },
  },
};
</script>
