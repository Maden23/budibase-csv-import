<script>
  // default slot does not work
  // how do i stop the import happing in preview - can i detect preview
  // API is not documented
  // documentation on how to trigger a defineActions?
  import { getContext } from "svelte";
  import Dropzone from "svelte-file-dropzone";
  import Papa from "papaparse";
  import { Table } from "@budibase/bbui";

  export let importToTable;
  export let table;
  export let additionalData="";
  export let importText="Import";
  export let exportErroredRowsText="Export errored rows"
  export let resetText="Reset";
  export let copyErrorText = "Copy errors"
  export let onImport= () => {};
  export let dragzoneText;

  const { styleable, Provider, API, builderStore } = getContext("sdk");
  const component = getContext("component");

  let file = {};
  let data = [];
  let schema = null;
  let parseErrors = [];
  let importErrors = [];
  let isParsed = false;
  let isReadyForPreview = false;
  let isImporting = false;
  let isImported = false;
  let importedCount = 0;
  $: hasRows = data.length > 0;

  $: dataContext = {
    data,
    file,
    parseErrors,
    importErrors,
    isParsed,
    isImported,
    isImporting,
    importedCount,
  };

  function handleFilesSelect(e) {
    const { acceptedFiles } = e.detail;
    if (acceptedFiles.length === 0) {
      file = null;
      return;
    }

    file = acceptedFiles[0];
    Papa.parse(file, {
      header: true,
      skipEmptyLines: true,
      complete: (result) => {
        data = result.data;
        parseErrors = result.errors;
        isParsed = true;
        if (additionalData !== "") {
          addUserData()
        }
        preparePreview();
      }
    });
  }

  function addUserData() {
    const userData = JSON.parse(additionalData);
    data = data.map( obj => ({...obj, ...userData}) );
  }

  async function preparePreview() {
    let tableDefinition = await API.fetchTableDefinition(table.tableId);
    schema = tableDefinition.schema
    isReadyForPreview = true;
  }

  async function importData() {
    isImporting = true;
    for (let row of data) {
      // do not import in builder mode
      if (importToTable && !$builderStore.inBuilder) {
        try {
          importedCount++;
          await API.saveRow({
            tableId: table.tableId,
            ...row
          });
        } catch(e) {
          importErrors.push({
            rowNumber:data.indexOf(row) + 1,
            error: e
          });
        }
      }
    }
    isImporting = false;
    isImported = true;
    onImport();
  }

  function reset() {
    data = [];
    file = null;
    parseErrors = [];
    importErrors = [];
    isParsed = false;
    isReadyForPreview = false;
    isImported = false;
    isImporting = false;
    importedCount = 0;
  }

  function copyErrors() {
    navigator.clipboard.writeText(JSON.stringify(importErrors));
  }

  function exportErroredRows() {
    var erroredRowNumbers = new Set();
    parseErrors.map(err => erroredRowNumbers.add(err.row));
    importErrors.map(err => erroredRowNumbers.add(err.rowNumber - 1));
    
    var erroredRows = []
    erroredRowNumbers.forEach(i => erroredRows.push(data[i]))

    var csv = Papa.unparse(erroredRows);
    var csvData = new Blob([csv], {type: 'text/csv;charset=utf-8;'});
    var csvURL =  null;
    if (navigator.msSaveBlob)
    {
        csvURL = navigator.msSaveBlob(csvData, 'errors.csv');
    }
    else
    {
        csvURL = window.URL.createObjectURL(csvData);
    }

    var tempLink = document.createElement('a');
    tempLink.href = csvURL;
    tempLink.setAttribute('download', 'errors.csv');
    tempLink.click();
  }

</script>
<div use:styleable={$component.styles} >
  {#if !isParsed}
    <Dropzone 
      on:drop={handleFilesSelect} 
      multiple=false>
      <p>{dragzoneText}</p>
    </Dropzone>
  {/if}
  <Provider data={dataContext}>
    <slot/>
    {#if !$component.children}
      <div class="info">
        {#if isParsed && !isImported}
          <p>{file.name}&nbsp;-&nbsp;{data.length}&nbsp;rows found</p>
        {/if}
        {#if isReadyForPreview}
          <Table {data} {schema} allowEditRows={false} allowEditColumns={false} allowSelectRows={false}></Table>
        {/if}
        {#if isImporting}
          <p>{importedCount}/{data.length}</p>
        {/if}
        {#if isImported}
          {#if importErrors.length === 0}
          <p>Import complete!</p>
            {:else}
            <div>
              {`${data.length - importErrors.length} Imported,  ${importErrors.length} Errored`}
            </div>
            {/if}
          {/if}
      </div>
    {/if}
  </Provider>

  <div class="buttons">
    {#if isParsed || importErrors.length > 0}
      <button 
        on:click={reset}
        class="spectrum-Button spectrum-Button--quiet spectrum-Button--secondary spectrum-Button--sizeS">
          {resetText}
      </button>
    {/if}
    {#if isParsed && importErrors.length > 0}
      <button 
        class="spectrum-Button spectrum-Button--fill spectrum-Button--primary spectrum-Button--sizeS"
        on:click={copyErrors}>
          {copyErrorText}
      </button>
      <button 
        class="spectrum-Button spectrum-Button--fill spectrum-Button--primary spectrum-Button--sizeS"
        on:click={exportErroredRows}>
          {exportErroredRowsText}
      </button>
    {/if}
    {#if isParsed && hasRows && !isImported && !isImporting}
      <button
        on:click={importData}
        class="spectrum-Button spectrum-Button--fill spectrum-Button--primary spectrum-Button--sizeS">
        {importText}
      </button>
    {/if}
  </div>

</div>

<style>
  .info {
    margin-left: 12px;
  }
  .buttons {
    margin-top: 10px;
  }
</style>
