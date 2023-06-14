<template>
  <v-app>
    <v-app-bar color="grey"
      ><v-app-bar-title
        ><v-icon icon="mdi-code-braces" color="#000080" class="pr-2"></v-icon>
        <span style="color: #000080"
          >Embedding API Examples</span
        ></v-app-bar-title
      ></v-app-bar
    >
    <v-main>
      <div class="px-6 pt-2 d-flex justify-center">
        <v-tabs color="#000080" v-model="selectedViz">
          <v-tab value="scatter"
            >Scatter - Profit vs. Discount by Customer</v-tab
          >
          <v-tab value="line">Line - Monthly Profit by Region</v-tab>
          <v-tab value="map">Map - Multiple Measures w/ Parameter</v-tab>
        </v-tabs>
      </div>
      <div class="d-flex justify-center align-start w-100 py-4 px-6">
        <div class="d-flex w-60" style="height: 725px">
          <div>
            <tableau-viz
              :src="activeSrcUrl"
              device="desktop"
              hide-tabs
              toolbar="hidden"
            >
            </tableau-viz>
          </div>
          <div
            class="h-100 d-flex justify-center align-center"
            v-if="activeSheetName != 'Scatter'"
          >
            <div
              v-if="activeSheetName == 'Map'"
              class="d-flex flex-column pl-4"
            >
              <span>Update a Parameter:</span>
              <v-select
                :items="
                  formattedParams &&
                  formattedParams[0] &&
                  formattedParams[0].allowableVals
                "
                item-title="value"
                item-value="value"
                variant="outlined"
                v-model="selectedParamVal"
                density="compact"
                @update:menu="
                  updateParameterValue('Dot Size', selectedParamVal)
                "
              >
              </v-select>
            </div>
            <div
              v-if="formattedFilters && activeSheetName == 'Line'"
              class="d-flex flex-column justify-center pl-4"
            >
              <span>Update Filters:</span>
              <v-checkbox
                v-model="selectedFilters"
                v-for="val in formattedFilters"
                :key="val"
                :label="val"
                :value="val"
                color="#000080"
                @change="updateFilters()"
              ></v-checkbox>
            </div>
          </div>
        </div>
      </div>
      <div v-if="marksFetched" class="d-flex flex-column px-4">
        <v-card
          elevation="2"
          v-if="formattedSelectedMarks && formattedSelectedMarks.length > 0"
        >
          <v-card-text>
            <v-data-table
              :headers="computedHeaders"
              :items="formattedSelectedMarks"
              item-value="name"
              density="compact"
              items-per-page="-1"
              hide-no-data=""
            >
              <template #bottom> </template>
              <template #top>
                <div class="d-flex justify-end align-center py-2">
                  <!-- Export:
                  <v-icon
                    icon="mdi-microsoft-excel"
                    size="32"
                    color="green"
                  ></v-icon> -->
                  Export
                  <json-c-s-v :data="formattedSelectedMarks">
                    <v-icon
                      icon="mdi-file-delimited-outline"
                      size="32"
                      color="#008000"
                      style="cursor: pointer"
                    ></v-icon>
                  </json-c-s-v></div
              ></template>
            </v-data-table>
          </v-card-text>
        </v-card>
      </div>
    </v-main>
  </v-app>
</template>

<script>
import { VDataTable } from "vuetify/labs/VDataTable";
import JsonCSV from "vue-json-csv";

export default {
  name: "App",

  data: () => ({
    selectedParamVal: "Order Count",
    embeddedViz: null,
    marksFetched: false,
    selectedMarks: [],
    activeSrcUrl:
      "https://10ax.online.tableau.com/t/kjmdev797388/views/EmbeddingWorkbook/Scatter",
    activeSheetName: "Scatter",
    selectedViz: "scatter",
    vizSheetMap: {
      scatter: {
        url: "https://10ax.online.tableau.com/t/kjmdev797388/views/EmbeddingWorkbook/Scatter",
        sheetName: "Scatter",
      },
      line: {
        url: "https://10ax.online.tableau.com/t/kjmdev797388/views/EmbeddingWorkbook/Line",
        sheetName: "Line",
      },
      map: {
        url: "https://10ax.online.tableau.com/t/kjmdev797388/views/EmbeddingWorkbook/Map",
        sheetName: "Map",
      },
    },
    wbParams: null,
    wsFilters: null,
    selectedFilters: ["West", "South", "East", "Central"],
  }),
  components: { VDataTable, JsonCSV },
  // computed properties allow of to transform things based on data in state
  // this is useful for dealing with complex promises/responses and turning them into
  // objects we want to interact with
  computed: {
    formattedFilters() {
      // Turn the complex wsFilters array into a simple array of the available filter values
      let filterVals = [];

      this.wsFilters &&
        this.wsFilters[0]._appliedValues.forEach((e) => {
          filterVals.push(e._value);
        });

      return filterVals;
    },
    formattedParams() {
      // Re-format the complex wbParams array into a list of objects with easy to read keys
      let paramArr = [];

      // this.wbParams && ... is a Vue JS notation to not evaluate/perform something until the first value is available
      // This helps prevent errors/invalid data when reactive properties are loading or haven't been set yet
      this.wbParams &&
        this.wbParams.forEach((e) => {
          if (e.parameterImpl._parameterInfo.name == "Dot Size") {
            let pObj = {
              name: e.parameterImpl._parameterInfo.name,
              currentValue: e.parameterImpl._parameterInfo.currentValue.value,
              allowableVals: e.parameterImpl._parameterInfo.allowableValues,
            };

            paramArr.push(pObj);
          }
        });

      return paramArr;
    },
    computedHeaders() {
      // Create table headers dynamically
      let headers = [];

      for (let col of this.formattedColNames) {
        let headerObj = {
          title: col,
          key: col,
        };
        headers.push(headerObj);
      }

      return headers;
    },
    // Format a new array of column names based on the selection marks
    formattedColNames() {
      let cols = this.selectedMarks._columns;

      return cols.map((e) => e._fieldName);
    },
    formattedSelectedMarks() {
      // Transform the selected marks data into a readable/tabular array of objects
      const marksData = [];

      this.selectedMarks._data.forEach((e) => {
        if (Array.isArray(e)) {
          marksData.push(e);
        }
      });

      const formattedMarks = [];

      for (let mark of marksData) {
        const formattedObj = {};

        for (let [index, value] of mark.entries()) {
          formattedObj[this.formattedColNames[index]] = value._value;
        }
        formattedMarks.push(formattedObj);
      }

      return formattedMarks;
    },
  },
  methods: {
    async initViz() {
      // This is required when the page loads and any time the embedded viz changes in order for it to be interactive!

      // Set the <tableau-viz> component to be our embedded viz for interactivity -- this all works because of the import added to index.html
      const viz = document.querySelector("tableau-viz");

      this.embeddedViz = viz;

      // Add an event listener to verify the viz becomes interactive
      await new Promise((resolve) => {
        this.embeddedViz.addEventListener("firstinteractive", () => {
          console.log("Viz is interactive!");
          resolve();
        });
      });

      // If we've selected the line sheet, query its filters
      if (this.activeSheetName === "Line") {
        this.getFilters();
      }

      // if we've selected the map sheet, query its parameters
      if (this.activeSheetName === "Map") {
        this.getParameterNames();
      }

      // Add another event listener so we know when marks selections change
      await new Promise((resolve) => {
        this.embeddedViz.addEventListener("markselectionchanged", () => {
          this.getSelections();
          resolve();
        });
      });
    },
    async getParameterNames() {
      // Get information about the parameters in the active workbook
      const viz = this.embeddedViz;

      this.wbParams = await viz.workbook.getParametersAsync();
    },

    async updateParameterValue(paramName, newValue) {
      // Update a parameter using its name and the new value
      const viz = this.embeddedViz;

      await viz.workbook.changeParameterValueAsync(paramName, newValue);
    },
    async getFilters() {
      // Get details about the filters for the activated sheet
      const viz = this.embeddedViz;

      const worksheet = await viz.workbook.activateSheetAsync(
        this.vizSheetMap.line.sheetName
      );

      this.wsFilters = await worksheet.getFiltersAsync();
    },
    async updateFilters() {
      // Update a filter in the activated sheet by passing its name and an array of values
      const viz = this.embeddedViz;

      const worksheet = await viz.workbook.activateSheetAsync(
        this.vizSheetMap.line.sheetName
      );

      // Stringify and then re-parse the selectedFilters array
      // This is necessary because of reactivity in Vue, i.e. it forces a value from what is otherwise a reference to a resolve Promise
      const filterArr = JSON.parse(JSON.stringify(this.selectedFilters));

      await worksheet.applyFilterAsync("Region", filterArr, "replace");
    },
    async getSelections() {
      // Get the selected marks from the active sheet (triggered by our eventListener in initViz())

      const viz = this.embeddedViz;
      const sheet = await viz.workbook.activateSheetAsync(this.activeSheetName);

      const selectedMarks = await sheet.getSelectedMarksAsync();

      // Map the data into a format for display, etc.
      this.selectedMarks = selectedMarks.data[0];
      this.marksFetched = true;
    },
  },
  watch: {
    // watchers in Vue allow us to perform an action when a value in state changes
    async selectedViz(nV, oV) {
      // When the selected viz changes, update information about the url and sheet name, then re-initialize
      if (nV != oV) {
        switch (nV) {
          case "line":
            this.activeSrcUrl = this.vizSheetMap.line.url;
            this.activeSheetName = this.vizSheetMap.line.sheetName;

            break;
          case "scatter":
            this.activeSrcUrl = this.vizSheetMap.scatter.url;
            this.activeSheetName = this.vizSheetMap.scatter.sheetName;

            break;

          case "map":
            this.activeSrcUrl = this.vizSheetMap.map.url;
            this.activeSheetName = this.vizSheetMap.map.sheetName;
        }

        // Ensure interactivity with the correct sheet when changed
        await this.initViz();
      }
    },
  },
  // Mounted actions occur when the component is added to the DOM
  async mounted() {
    // On page load, initialize the first viz
    this.initViz();
  },
};
</script>
