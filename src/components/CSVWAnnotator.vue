<template>
  <v-container>
    <v-row>
      <v-col>
        <h1>CSV annotator</h1>
      </v-col>
    </v-row>
    <v-form>
      <v-row>
        <v-col md="6">
          <v-file-input
            accept="text/csv"
            label="CSV file"
            v-model="csvFile"
          ></v-file-input>
        </v-col>
        <v-col md="2">
          <v-select
            label="Delimiter"
            :items="delimiters"
            v-model="csvOptions.delimiter"
          >
          </v-select>
        </v-col>
        <v-col md="2">
          <v-autocomplete
            label="Encoding"
            :items="csvEncodings"
            v-model="csvEncoding"
          ></v-autocomplete>
        </v-col>
        <v-col md="2">
          <v-text-field
            type="number"
            label="Preview rows"
            v-model="csvOptions.previewRows"
            min="1"
          ></v-text-field>
        </v-col>
      </v-row>
      <v-row class="mt-0">
        <v-col class="text-center">
          <v-btn @click.prevent="loadFile">Load file</v-btn>
        </v-col>
      </v-row>
    </v-form>
    <v-row v-if="headers">
      <v-col>
        <v-card>
          <v-tabs
            align-tabs="center"
            v-model="tab"
            :items="tabs"
          >
          </v-tabs>
          <v-card-text>
            <v-tabs-window v-model="tab">
              <v-tabs-window-item value="preview">
                <v-data-table
                  :items="previewTableItems"
                >
                </v-data-table>
              </v-tabs-window-item>

              <v-tabs-window-item value="bindings">
                <v-data-table
                  :items="headers"
                  :headers="bindingHeaders"
                >
                  <template #item.concept="{ item }">
                    <div class="d-flex flex-nowrap">
                      <v-autocomplete
                        density="compact"
                        hide-details
                        class="mx-1 flex-grow-1"
                        label="Term collection"
                        :items="conceptSchemes.values"
                        v-model="item.conceptScheme"
                        @update:model-value="updateHeaderConceptScheme(item)"
                        return-object
                      ></v-autocomplete>
                      <v-progress-circular
                        v-if="item.conceptScheme?.loading"
                        size="16"
                        indeterminate
                      ></v-progress-circular>
                      <v-autocomplete
                        density="compact"
                        hide-details
                        v-if="item.conceptScheme?.concepts"
                        class="mx-1 flex-grow-1"
                        label="Term"
                        :items="item.conceptScheme.concepts"
                        v-model="item.concept"
                      ></v-autocomplete>
                    </div>
                  </template>

                  <template #item.datatype="{ item }">
                    <v-autocomplete
                        density="compact"
                        hide-details
                        class="mx-1"
                        label="Data type"
                        :items="csvwDataTypes"
                        v-model="item.datatype"
                      ></v-autocomplete>
                  </template>
                </v-data-table>
              </v-tabs-window-item>

              <v-tabs-window-item value="result">
                <pre>{{ csvwResult }}</pre>
              </v-tabs-window-item>
            </v-tabs-window>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>
<script>
import {parse} from 'csv-parse/browser/esm';

const sparqlEndpoint = import.meta.env.VITE_SPARQL_ENDPOINT;

const csvwDataTypes = [
  'anyURI',
  'base64Binary',
  'binary',
  'boolean',
  'byte',
  'date',
  'dateTime',
  'dateTimeStamp',
  'datetime',
  'dayTimeDuration',
  'decimal',
  'double',
  'duration',
  'float',
  'gDay',
  'gMonth',
  'gMonthDay',
  'gYear',
  'gYearMonth',
  'hexBinary',
  'int',
  'integer',
  'language',
  'long',
  'negativeInteger',
  'nonNegativeInteger',
  'nonPositiveInteger',
  'normalizedString',
  'number',
  'positiveInteger',
  'short',
  'string',
  'time',
  'token',
  'unsignedByte',
  'unsignedInt',
  'unsignedLong',
  'unsignedShort',
  'yearMonthDuration',
]

const sparqlQuery = async (query) => {
  return await fetch(sparqlEndpoint, {
    method: 'POST',
    body: query,
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/sparql-query',
    },
  });
};

export default {
  data() {
    const csvEncodings = ['utf-8'];
    for (let i = 1; i <= 16; i++) {
      csvEncodings.push(`iso-8859-${i}`);
    }
    return {
      csvFile: null,
      csvOptions: {
        delimiter: ',',
        previewRows: 10,
      },
      csvEncoding: 'utf-8',
      csvEncodings,
      csvError: null,
      delimiters: [
        {title: 'Comma (,)', value: ','},
        {title: 'Semicolon (;)', value: ';'},
      ],
      loadingCsvFile: false,
      dataRows: [],
      tab: 'preview',
      tabs: [
        {text: 'Preview data', value: 'preview'},
        {text: 'Bindings', value: 'bindings'},
        {text: 'Result', value: 'result'},
      ],
      conceptSchemes: {
        loading: false,
        values: null,
      },
      headers: null,
      bindingHeaders: [
        { title: 'Header', value: 'text', width: "20%"},
        { title: 'Concept', value: 'concept' },
        { title: 'Datatype', value: 'datatype' },
      ],
      csvwDataTypes,
    };
  },
  mounted() {
    this.loadConceptSchemes();
  },
  computed: {
    previewTableItems() {
      if (!this.headers) {
        return [];
      }
      return this.dataRows.map(d => Object.fromEntries(this.headers.map((h, i) => [h.text, d[i]])));
    },
    csvwResult() {
      if (!this.headers || !this.conceptSchemes.values) {
        return null;
      }
      const columns = this.headers.map(header => {
        const mapping = {
          name: header.text,
        };
        if (header.concept) {
          mapping.propertyUrl = header.concept;
        }
        if (header.datatype) {
          mapping.datatype = header.datatype;
        }
        return mapping;
      });
      const result = {
        '@context': "http://www.w3.org/ns/csvw",
        tableSchema: {
          columns,
        },
        dialect: {
          header: true,
        },
      };
      return result;
    },
  },
  methods: {
    async loadFile() {
      if (!this.csvFile) {
        return;
      }
      this.loadingCsvFile = true;
      this.headers = null;
      this.dataRows = [];

      let csvDone = false;
      const parser = parse(Object.assign(
        {to: this.csvOptions.previewRows + 1},
        this.csvOptions
      ));
      parser.on('readable', () => {
        let record;
        while ((record = parser.read()) !== null) {
          if (!this.headers) {
            this.headers = record.map(r => ({
              text: r,
              conceptScheme: null,
              concept: null,
              datatype: null,
            }));
          } else {
            this.dataRows.push(record);
          }
        }
      });
      parser.on('error', (err) => {
        this.csvError = err;
        csvDone = true;
      });
      parser.on('end', () => {
        csvDone = true;
      });
      parser.destroy = () => {
      };

      const reader = await this.csvFile.stream().getReader();
      const decoder = new TextDecoder(this.csvEncoding);
      while (!csvDone) {
        const {done, value} = await reader.read();
        if (done || csvDone) {
          break;
        }
        const text = decoder.decode(value);
        parser.write(text);
      }
      parser.end();
    },
    updateHeaderConceptScheme(item) {
      item.concept = null;
      this.loadConcepts(item.conceptScheme);
    },
    async loadConceptSchemes() {
      const query = `
        PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
        SELECT DISTINCT ?cs ?label WHERE { ?cs a skos:ConceptScheme ; skos:prefLabel ?label }
        ORDER BY ?label
      `;
      const response = await sparqlQuery(query);
      const data = await response.json();
      if (data?.results?.bindings?.length) {
        this.conceptSchemes.values = data.results.bindings.map(b => ({
          title: b.label.value,
          value: b.cs.value,
          loading: false,
          concepts: null,
        }));
      }
    },
    async loadConcepts(conceptScheme) {
      if (conceptScheme.loading || conceptScheme.concepts) {
        return;
      }
      conceptScheme.loading = true;
      try {
        const query = `
          PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
          SELECT DISTINCT ?c ?label WHERE { ?c a skos:Concept ; skos:prefLabel ?label ; skos:inScheme <${conceptScheme.value}> }
          ORDER BY ?label
        `;
        const response = await sparqlQuery(query);
        const data = await response.json();
        if (data?.results?.bindings?.length) {
          conceptScheme.concepts = data.results.bindings.map(b => ({
            title: b.label.value,
            value: b.c.value,
          }));
        }
      } finally {
        conceptScheme.loading = false;
      }
    },
  },
}
</script>
