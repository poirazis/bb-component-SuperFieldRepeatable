<script>
  import { getContext } from "svelte";
  import { SuperCell } from "../../bb_super_components_shared/src/lib"
  import { dndzone } from "svelte-dnd-action";

  const { styleable, builderStore, componentStore } = getContext("sdk");
  const component = getContext("component");

  const formContext = getContext("form");
  const formStepContext = getContext("form-step");
  const labelPos = getContext("field-group");
  const labelWidth = getContext("field-group-label-width");
  const formApi = formContext?.formApi;

  export let stringField;
  export let numberField;
  export let booleanField;
  export let datetimeField;
  export let optionsField;
  export let linkField;
  export let arrayField;
  export let stringValidation;
  export let numberValidation;
  export let booleanValidation;
  export let datetimeValidation;
  export let optionsValidation;
  export let linkValidation;
  export let arrayValidation;

  export let frontButtons = [];
  export let frontButtonsQuiet;
  export let frontButtonsGrouping;
  export let frontButtonsSelected = [0];
  export let buttons = [];
  export let buttonsQuiet;
  export let buttonsGrouping;
  export let buttonsSelected = [0];

  export let fieldLabel;
  export let fieldType = "string";
  export let span = 6;
  export let inForm = false;
  export let repeatable = false
  export let minEntries = 1
  export let maxEntries = 10
  export let reordering
  export let placeholder
  export let defaultValue

  let formField;
  let formStep;
  let fieldState;
  let fieldApi;
  let value;

  let wrapperAnchor;
  let items = []
  let flipDurationMs = 130


  $: initItems( repeatable ? minEntries : 1 )
  $: dragDisabled = !reordering

  $: fieldName =
    fieldType == "string"
      ? stringField
      : fieldType == "number"
        ? numberField
        : fieldType == "boolean"
          ? booleanField
          : fieldType == "datetime"
            ? datetimeField
            : fieldType == "options"
              ? optionsField
              : fieldType == "link"
                ? linkField
                : fieldType == "array"
                  ? arrayField
                  : stringField;

  $: formStep = formStepContext ? $formStepContext || 1 : 1;
  $: formField = formApi?.registerField(
    fieldName,
    fieldType,
    defaultValue,
    false,
    null,
    formStep
  );
  $: value = fieldState?.value ? fieldState.value : defaultValue
  $: disabled = fieldState?.disabled;

  $: unsubscribe = formField?.subscribe((value) => {
    fieldState = value?.fieldState;
    fieldApi = value?.fieldApi;
  });

  $: {
    if (
      $builderStore.inBuilder &&
      $componentStore.selectedComponentPath?.includes($component.id) &&
      formContext &&
      !inForm
    ) {
      builderStore.actions.updateProp("inForm", true);
    } else if (
      $builderStore.inBuilder &&
      $componentStore.selectedComponentPath?.includes($component.id) &&
      inForm &&
      !formContext
    ) {
      builderStore.actions.updateProp("inForm", false);
    }
  }

  $: $component.styles = {
    ...$component.styles,
    normal: {
      ...$component.styles.normal,
      "flex-direction": labelPos == "left" ? "row" : "column",
      "min-height": items?.length == 1 ? labelPos == "left" ? "2rem" : "3.75rem"
                                       : labelPos == "left" ? ((items?.length * 2.25) )+ "rem"
                                       : ( (items?.length * 2.25) + 1.75 ) + "rem",

      gap: labelPos == "left" ? "0.85rem" : "0rem",
      "grid-column": labelPos ? "span " + span : null,
      "--label-width":
        labelPos == "left" ? (labelWidth ? labelWidth : "6rem") : "auto",
    },
  };
 
  function handleDndConsider(e) {
    items = e.detail.items;
  }

  function handleDndFinalize(e) {
    items = e.detail.items;
  }

  const initItems = ( num ) => {
    items = []
    for (let i = 1; i <= num; i++ )
    {
      items.push({id: Math.random() })
    }
    items = items
  }

  const handleKeyboard = ( e, idx ) => {
    if ( e.key == "Enter" && repeatable ) {
      if ( items[idx].value && idx == items.length - 1 ) {
        buttonClick("repeater_add")
        setTimeout( () => items.at(-1)?.cellState?.focus(), 10)
      } else if (items[idx].value ) {
        items[idx].cellState.unfocus()
      }
    }

    if ( e.key == "Escape") {
      items[idx].cellState.unfocus() 
      if ( !items[idx].value )
        buttonClick("repeater_remove", idx)
    }

    if ( e.key == "ArrowDown") {
      if ( idx < items.length - 1 )
        items[idx+1].cellState.focus()
      else
        items[0].cellState.focus();
    }

    if ( e.key == "ArrowUp") {
      if ( idx > 0 )
        items[idx - 1].cellState.focus()
      else 
        items.at(-1).cellState.focus()
    }
  }

  const buttonClick = (group, idx) => {
    if (group == "front" && frontButtonsGrouping == "single") {
      frontButtonsSelected = [idx];
    } else if (group == "end" && buttonsGrouping == "single") {
      buttonsSelected = [idx];
    }

    if (group == "front" && frontButtonsGrouping == "multi") {
      let i = frontButtonsSelected.indexOf(idx);
      if (i > -1) frontButtonsSelected.splice(i, 1);
      else frontButtonsSelected.push(idx);

      frontButtonsSelected = frontButtonsSelected;
    } else if (group == "end" && buttonsGrouping == "multi") {
      buttonsSelected = [idx];
    }

    if ( group == "repeater_add" ) {
      if (items.at(-1).value && items.length < maxEntries) {
        items = [...items, { id: Math.random(), value: undefined} ]
        setTimeout( () => items.at(-1)?.cellState?.focus(), 10)
      }
      else {
        items.at(-1).cellState.focus();
      } 
    }

    if ( group == "repeater_remove" ) {
      if ( items.length > minEntries ) {
        items.splice(idx,1)
        items = items;
      }
    }
  };

  $: console.log( items[0].cellState )
</script>

<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-noninteractive-tabindex -->
<div
  class="superField"
  bind:this={wrapperAnchor}
  on:focus={() => items.at(0)?.cellState?.focus() } 
  tabindex="0"
  use:styleable={$component.styles}  
>
  <label for="superCell" class="superFieldLabel" class:bound={formContext}>
    {fieldLabel || fieldName || "Unamed Field"}
  </label>
  
  <div class="cell-items" >

    {#each items as item , item_idx (item.id)}
      <div class="inline-cell" on:keydown={(e) => handleKeyboard (e, item_idx)} >
        
        {#if repeatable && reordering }
          <div class="dragHandle">
            <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-grip-vertical"><circle cx="9" cy="12" r="1"/><circle cx="9" cy="5" r="1"/><circle cx="9" cy="19" r="1"/><circle cx="15" cy="12" r="1"/><circle cx="15" cy="5" r="1"/><circle cx="15" cy="19" r="1"/></svg>
          </div>
        {/if}

        {#if frontButtons?.length}
          <div
            class="spectrum-ActionGroup spectrum-ActionGroup--compact spectrum-ActionGroup--sizeM"
            class:spectrum-ActionGroup--quiet={frontButtonsQuiet}
          >
            {#each frontButtons as button, idx}
              <button
                on:click={(e) => buttonClick("front", idx)}
                class="spectrum-ActionButton spectrum-ActionButton--sizeM"
                class:spectrum-ActionButton--quiet={frontButtonsQuiet ||
                  button.quiet}
                class:warning={button.type == "warning"}
                class:cta={button.type == "cta"}
                class:disabled={button.disabled}
                class:is-selected={frontButtonsGrouping &&
                  frontButtonsSelected.includes(idx)}
              >
                <span class="spectrum-ActionButton-label">{@html button?.text}</span
                >
              </button>
            {/each}
          </div>
        {/if}

        <div class="inline-cells" on:mousedown|preventDefault|stopPropagation={ () => setTimeout( () => items.at(item_idx)?.cellState?.focus(), 10) } >
          <SuperCell
            bind:cellState={items[item_idx].cellState}
            cellOptions={{ placeholder, defaultValue, role: "formInput" }}
            value={ repeatable ? items[item_idx].value : value }
            fieldSchema={{ type: fieldType }}
            editable
            id={"sp_cell_" + item_idx}
            on:change={(e) => items[item_idx].value = e.detail}
            on:blur={() => items.at(item_idx)?.cellState?.lostFocus()}
          />
        </div>

        {#if buttons?.length}
          <div
            class="spectrum-ActionGroup spectrum-ActionGroup--compact spectrum-ActionGroup--sizeM"
            class:spectrum-ActionGroup--quiet={buttonsQuiet}
          >
            {#each buttons as button, idx}
              <button
                class="spectrum-ActionButton spectrum-ActionButton--sizeM"
                on:click={(e) => buttonClick("end", idx)}
                class:spectrum-ActionButton--quiet={buttonsQuiet || button.quiet}
                class:warning={button.type == "warning"}
                class:cta={button.type == "cta"}
                class:disabled={button.disabled}
                class:is-selected={buttonsGrouping && buttonsSelected.includes(idx)}
              >
                <span class="spectrum-ActionButton-label">{button.text}</span>
              </button>
            {/each}
          </div>
        {/if}

        {#if repeatable}
          <div
            class="spectrum-ActionGroup spectrum-ActionGroup--compact spectrum-ActionGroup--sizeS"
            class:spectrum-ActionGroup--quiet={true}
          >
            {#if items.length == 1 || item_idx == (items.length - 1) }
              <button
                tabindex="-1"
                class="spectrum-ActionButton spectrum-ActionButton--sizeS"
                on:mousedown|preventDefault={(e) => buttonClick("repeater_add", item_idx)}
                class:spectrum-ActionButton--quiet={true}
              >
                <span class="spectrum-ActionButton-label">
                  <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-plus"><path d="M5 12h14"/><path d="M12 5v14"/></svg>
                </span>
              </button>
            {/if}

            {#if items.length > 1 }
              <button
              tabindex="-1"
                class="spectrum-ActionButton spectrum-ActionButton--sizeS delete"
                on:click={(e) => buttonClick("repeater_remove", item_idx)}
                class:spectrum-ActionButton--quiet={true}
              >
                <span class="spectrum-ActionButton-label">
                  <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-x"><path d="M18 6 6 18"/><path d="m6 6 12 12"/></svg>
                </span>
              </button>
            {/if}

          </div>
        {/if}

      </div>
    {/each}
  </div>

</div>

<style>
  .superField {
    display: flex;
    align-items: stretch;
    justify-content: stretch;
    min-width: 0;
  }

  .superField:focus {
    outline: none;
  }
  .superFieldLabel {
    display: flex;
    align-items: flex-start;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    min-width: var(--label-width);
    max-width: var(--label-width);
    font-size: 14px;
    line-height: 1.75rem;
    font-weight: 400;
    color: var(--spectrum-global-color-gray-700);
  }

  .cell-items {
    flex: 1;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    align-items: stretch;
  }

  .inline-cell {
    flex: 1;
    display: flex;
    flex-direction: row;
    align-items: center;
    gap: 0.25rem;
  }
  .inline-cell:focus {
    outline: none;
  }

  .inline-cells {
    flex: 1;
    display: flex;
    justify-items: stretch;
    height: 2rem;
  }
  .warning {
    color: var(--spectrum-global-color-red-500);
    border-color: var(--spectrum-global-color-red-500);
  }

  .spectrum-ActionButton--quiet.warning {
    color: var(--spectrum-global-color-red-500);
    border: none;
  }
  .spectrum-ActionButton--quiet.delete {
    color: var(--spectrum-global-color-red-500);
  }

  .spectrum-ActionButton--quiet.delete:hover{
    border-color:  var(--spectrum-global-color-red-500);
  }

  .warning:hover {
    color: #000;
    background-color: var(--spectrum-global-color-red-500);
  }
  .cta {
    color: #fff;
    background-color: var(--primaryColor);
  }

  .disabled {
    color: var(--spectrum-global-color-gray-600);
    background-color: var(--spectrum-global-color-gray-200);
  }

  :global(.spectrum-ActionGroup > button > span > svg) {
    width: 14px;
    height: 14px;
  }

  .superFieldLabel.bound {
    gap: 0.5rem;
  }

  .dragHandle {
    flex: 0;
    display: flex;
    align-items: center;
    height: 100%;
    color: var(--spectrum-global-color-gray-500);
  }
  .dragHandle:hover {
    color: var(--spectrum-global-color-gray-900);
  }
</style>
