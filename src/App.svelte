<script>
	import { onMount } from 'svelte';
	import { fly } from 'svelte/transition';
	import { cubicInOut } from 'svelte/easing';
	import { Color, MeshLambertMaterial } from "three";
	import { IfcViewerAPI } from "web-ifc-viewer";
	import Icon from './Icon.svelte';
	import Help from './Help.svelte';

	let fileInput;
	let container;
	let viewer;
	let propertyData;

	let helpActive;
	let selectionActive = true;
	let hideActive = false;
	let isolateActive = false;
	let clipperActive = false;
	let transparentActive = false;
	let detailsActive = false;
	let measuresActive = false;
	let disabledButtons = true;

	let wsObject;
	let activeModel;
	let workstations;
	let workstation = 'All';

	let selectedIDs = [];
	let selectedSubset;
	let subsets = {};

	const transparentMat = new MeshLambertMaterial({
		transparent: true,
		opacity: 0.2,
		color: 0xffffff,
		depthTest: true,
	});


	onMount(async () => {
	//------------------- Creates the viewer -------------------
		viewer = setupScene(container);
	})
	
	function setupScene(container) {
	//------------------- Sets up ThreeJS scene -------------------
		const viewer = new IfcViewerAPI({
			container, backgroundColor: new Color(0xfffff8),
		});
		viewer.IFC.applyWebIfcConfig({
			COORDINATE_TO_ORIGIN: true,
      		USE_FAST_BOOLS: true,
    	})
		viewer.IFC.setWasmPath("wasm/");
		return viewer;
	}

	const loadIfc = async (event) => {
	//------------------- Loading IFC -------------------
		const model = await viewer.IFC.loadIfc(event.target.files[0]);

		let JSONblob = await viewer.IFC.properties.serializeAllProperties(model);
		let JSONdata = await JSONblob[0].text();
		JSONdata = JSON.parse(JSONdata);
		const results = await createWsArray(JSONdata, model);
		[workstations, wsObject] = results;

		createSubsets(viewer, model, wsObject);
		replaceModelBySubset(viewer, model, "All");

		propertyData = await getItemProperties(model.modelID);
		disabledButtons = false;
	}

	function handleKeyPress(e) {
	//------------------- Handles key presses -------------------
		switch (e.code) {
			case 'Escape':
				viewer.clipper.active && viewer.clipper.deleteAllPlanes();
				if (measuresActive) {
					measuresActive = !measuresActive;
					viewer.dimensions.previewActive = measuresActive;
					viewer.dimensions.deleteAll();
				}
				break;
			case 'KeyV':
				toggleHide(workstation);
				break;
			case 'KeyI':
				toggleIsolate(workstation);
				break;
			case 'KeyT':
				toggleTransparent(workstation);
				break;
			case 'KeyD':
				toggleDetails();
				break;
			case 'KeyC':
				toggleClipper();
				break;
			case 'KeyM':
				toggleDimensions();
				break;
			case 'KeyH':
				toggleHelp();
				break;
		};
	}

	function handleDoubleClick() {
		if (viewer.clipper.active) {
			viewer.clipper.createPlane();
		}	else if (measuresActive) {
			viewer.dimensions.create();
		}
	}

	async function prePickItem() {
	//------------------- Pre-picks item -------------------
		if (selectionActive) {
			await viewer.IFC.selector.prePickIfcItem();
		}

	}

	async function pickItem() {
	//------------------- Handles item picking -------------------
		if (viewer && selectionActive) {
			const found = await viewer.IFC.selector.pickIfcItem();
			if (found) {
				propertyData = await getItemProperties(found.id);
				if (isolateActive) {
					selectedIDs = [found.id];
					isolate(selectedIDs);
				}
				if (hideActive) {
					selectedIDs.push(found.id);
					hide(selectedIDs);
					viewer.IFC.selector.unpickIfcItems();
				}
			} else {
				viewer.IFC.selector.unpickIfcItems();
			};
		};
	}

	function toggleHide(ws) {
	//------------------- Hide Button -------------------
		hideActive = !hideActive;
		if (!hideActive) {
			if (selectedSubset) {
				viewer.IFC.loader.ifcManager.clearSubset(activeModel.modelID, 'hidden');
				viewer.IFC.selector.unpickIfcItems();
				replaceModelBySubset(viewer, activeModel, ws);
			}
		}
		else {
			isolateActive && toggleIsolate(ws);
		};
	}

	function toggleIsolate(ws) {
	//------------------- Isolate Button -------------------
		isolateActive = !isolateActive;
		if (!isolateActive) {
			if (selectedSubset) {
				viewer.IFC.loader.ifcManager.clearSubset(activeModel.modelID, 'selected');
				viewer.IFC.selector.unpickIfcItems();
				replaceModelBySubset(viewer, activeModel, ws);
			}
		}
		else {
			hideActive && toggleHide(ws);
		};
	}

	function toggleClipper() {
	//------------------- Plane Button -------------------
		clipperActive = !clipperActive;
		viewer.clipper.active = clipperActive;
	}

	function toggleDetails() {
	//------------------- Details Button -------------------
		detailsActive = !detailsActive;
	}

	function toggleTransparent(ws) {
	//------------------- Transparency Button -------------------
		transparentActive = !transparentActive;
		transparentActive ? showTransparentWS(ws) : showTransparentWS('All');
	}

	function toggleDimensions() {
	//------------------- Dimensions Button -------------------
		if(measuresActive) {
			measuresActive = !measuresActive;
			viewer.dimensions.deleteAll();
			viewer.dimensions.previewActive = measuresActive;
			selectionActive = !measuresActive;
		} else {
			measuresActive = !measuresActive;
			viewer.dimensions.active = measuresActive;
			viewer.dimensions.previewActive = measuresActive;
			viewer.IFC.selector.unPrepickIfcItems();
			viewer.IFC.selector.unpickIfcItems();
			selectionActive = !measuresActive;
		};
	}

	function toggleWorkstation(ws) {
	//------------------- Workstation Button -------------------
		viewer.IFC.selector.unpickIfcItems();
		if (selectedSubset) {
			try {
				viewer.IFC.loader.ifcManager.clearSubset(activeModel.modelID, 'selected');
			} catch (e) {}
			try {
				viewer.IFC.loader.ifcManager.clearSubset(activeModel.modelID, 'hidden');
			} catch (e) {}
		};
		replaceModelBySubset(viewer, activeModel, ws);
		transparentActive && showTransparentWS(ws);
		isolateActive = false;
		hideActive = false;
	}

	function toggleHelp() {
	//------------------- Help Button -------------------
		helpActive = !helpActive;
		console.log('help')
	}

	async function createWsArray(JSONdata, model) {
	//------------------- Creates arrays of workstations and IDs -------------------
		let elementProp = [];
		let workStations = [];

		for (var id in JSONdata) {
			if (JSONdata[id]['type'] === 'IFCRELDEFINESBYPROPERTIES') {
				let elementID = JSONdata[id]['RelatedObjects'][0];
				let propSetID = JSONdata[id]['RelatingPropertyDefinition'];
				elementProp.push({[elementID] : propSetID});
			};
		};

		const allIDs = Array.from(new Set(model.geometry.attributes.expressID.array),);
		let objectWS = {'All': allIDs};

		for (var key in elementProp) {
			let obj = Object.keys(elementProp[key])[0];
			let propSet = JSONdata[elementProp[key][obj]];
			for (var Sprops in propSet['HasProperties']) {
				if (JSONdata[propSet['HasProperties'][Sprops]]['Name'] === 'Station') {
					let propSetValue = JSONdata[propSet['HasProperties'][Sprops]]['NominalValue'];
					objectWS.hasOwnProperty(propSetValue) ?	objectWS[propSetValue].push(parseInt(obj)) : objectWS[propSetValue] = [parseInt(obj)];
					!workStations.includes(propSetValue) &&	workStations.push(propSetValue);
				};
			};
		};
		return [workStations, objectWS];
	}

	function createSubsets(viewer, model, objects) {
	//------------------- Creates subsets -------------------
		const scene = viewer.context.getScene();

		for (const key in objects) {
			const subset = viewer.IFC.loader.ifcManager.createSubset({
				modelID: model.modelID,
				ids: objects[key],
				scene: scene,
				removePrevious: true,
				customID: key,
			});
			subsets[key] = subset;
		};
	}

	function replaceModelBySubset(viewer, model, id) {
	//------------------- Replaces current model with subset -------------------
		const items = viewer.context.items;
		const subset = subsets[id];

		for (const key in subsets) {
			key !== id && subsets[key].removeFromParent();
		}
		model.removeFromParent();
		items.ifcModels = [subset];

		activeModel = items.ifcModels[0];
		togglePickable(subset, true);
		viewer.context.scene.add(subset);
	}

	function hide(ids){
	//------------------- Hides the selected element(s) -------------------
		let currentIds = wsObject[workstation];
		let newIds = currentIds.filter(x => !ids.includes(x));
		const scene = viewer.context.getScene();
		activeModel.removeFromParent();
		selectedSubset = viewer.IFC.loader.ifcManager.createSubset({
			modelID: activeModel.modelID,
			ids: newIds,
			scene: scene,
			removePrevious: true,
			customID: 'hidden',
		});
		togglePickable(selectedSubset, true);	
	}

	function isolate(ids) {
	//------------------- Isolates selected item -------------------
		const scene = viewer.context.getScene();
		activeModel.removeFromParent();
		selectedSubset = viewer.IFC.loader.ifcManager.createSubset({
			modelID: activeModel.modelID,
			ids: ids,
			scene: scene,
			removePrevious: true,
			customID: 'selected',
		});
		togglePickable(selectedSubset, true);
	}

	function togglePickable(subset, pickable) {
	//------------------- Toggles pickable state of subset -------------------
		const items = viewer.context.items;
		pickable ? items.pickableIfcModels = [subset] : items.pickableIfcModels = activeModel;
	}

	function showTransparentWS(ws) {
	//------------------- Shows transparent workstations -------------------
		const scene = viewer.context.getScene();
		let wsIndex = workstations.indexOf(ws);
		let wsBefore = workstations.slice(0, wsIndex);
		let wsBeforeIDs = [];
		if (ws !== 'All') {
			for (var i = 0; i < wsBefore.length; i++) {
				wsBeforeIDs = wsBeforeIDs.concat(wsObject[wsBefore[i]]);
			};
		};

		let transparentSubset = viewer.IFC.loader.ifcManager.createSubset({
			modelID: activeModel.modelID,
			ids: wsBeforeIDs,
			scene: scene,
			removePrevious: true,
			customID: 'transparent',
			material: transparentMat,
		});
		scene.add(transparentSubset);
	}

	async function getItemProperties(expID) {
	//------------------- Gets prop data of ID -------------------
		const props = await viewer.IFC.getProperties(0, expID, true);
		const data = await getPropertyGroups(props);
		return data;
	}

	async function getPropertyGroups(values) {
	//------------------- Gets S parameters of item -------------------
		const props = await values;
		const properties = await viewer.IFC.getProperties(activeModel.modelID, props.expressID, true, true);

		if (props.psets.length === 0) {
			return {name: getProp(props.LongName), description: props.constructor.name, props: [{ name: 'Name', value: getProp(props.LongName)}, { name: 'GlobalID', value: getProp(props.GlobalId)}]};
		} else {
			const propObj = await properties['psets'][0]['HasProperties'].map((p) => {
				return { name: getProp(p.Name), value: getProp(p.NominalValue) };
			});
			try {
				propObj.splice(16, 1);
				propObj.splice(9, 6);
			} catch(e) {}
			return {name: props.Name.value, description: props.constructor.name, props: propObj};
		};
	}

	function getProp(prop) {
	//------------------- Handles specific prop -------------------
		if (prop == null || prop == undefined) return 'undefined';
		if (prop.value != undefined) return '' + prop.value;
		if (typeof prop == 'number') return '' + prop;
		return 'undefined';
	}

</script>

<main>
	<div id="viewer-container"
		bind:this={container}
		on:click={pickItem}
		on:keydown={handleKeyPress}
		on:mousemove={prePickItem}
		on:dblclick={handleDoubleClick}>
	</div>
	<aside class="toolbar">
		<input style="display:none" type="file" accept=".ifc" on:change={(e)=>loadIfc(e)} bind:this={fileInput} >
		<button class='button' class:non-active="{!disabledButtons}" on:click={()=>{fileInput.click()}} on:keydown={handleKeyPress}>
			<Icon name='upload' class='icon'/>
			<span class='tooltip'>Upload een IFC bestand</span>
		</button>
		<button class='button' class:selected="{hideActive}" class:non-active="{disabledButtons}" on:click={toggleHide(workstation)} on:keydown={handleKeyPress}>
			<Icon name='hide' class='icon'/>
			<span class='tooltip'><p>Verberg elementen</p></span>
		</button>
		<button class='button' class:selected="{isolateActive}" class:non-active="{disabledButtons}" on:click={toggleIsolate(workstation)} on:keydown={handleKeyPress}>
			<Icon name='isolate' class='icon'/>
			<span class='tooltip'><p>Isoleer een element</p></span>
		</button>
		<button class='button' class:selected="{clipperActive}" class:non-active="{disabledButtons}" on:click={toggleClipper} on:keydown={handleKeyPress}>
			<Icon name='section' class='icon'/>
			<span class='tooltip'><p>Creëer doorsnedes</p></span>
		</button>
		<button class='button' class:selected="{measuresActive}" class:non-active="{disabledButtons}" on:click={toggleDimensions} on:keydown={handleKeyPress}>
			<Icon name='measure' class='icon'/>
			<span class='tooltip'><p>Meten van elementen</p></span>
		</button>
		<button class='button' class:selected="{transparentActive}" class:non-active="{disabledButtons}" on:click={toggleTransparent(workstation)} on:keydown={handleKeyPress}>
			<Icon name='transparent' class='icon'/>
			<span class='tooltip'><p>Transparante vorige stations</p></span>
		</button>
		<button class='button' class:selected="{detailsActive}" class:non-active="{disabledButtons}" on:click={toggleDetails} on:keydown={handleKeyPress}>
			<Icon name='details' class='icon'/>
			<span class='tooltip'><p>Details van het element</p></span>
		</button>
		<button class='button' class:selected="{helpActive}" on:click={toggleHelp} on:keydown={handleKeyPress}>
			<Icon name='help' class='icon'/>
			<span class='tooltip'><p>Hulp voor de viewer</p></span>
		</button>
	</aside>
	{#if detailsActive}
	<div class="side-menu-right" transition:fly="{{ x: '100%', easing: cubicInOut}}">
		<div class="right-menu-item">
		    {#await propertyData then pset}
				<h4>{pset.name}</h4>
				<i id='description'>{pset.description}</i><br/>
				<hr/>
				<table>
					<thead>
						<tr>
							<th id='col1'>Eigenschap</th>
							<th id='col2'>Waarde</th>
						</tr>
					</thead>
					<tbody>
						{#each pset.props as row}
							<tr>
								<td id='col1'>{row.name}</td>
								<td id='col2'>{row.value}</td>
							</tr>
						{/each}
					</tbody>
				</table>
			{/await}	
		</div>
	</div>
	{/if}
	{#if (activeModel != null)}
		<div class='wsbar' in:fly="{{ y: '100%', easing: cubicInOut}}">
			<label class='button'>
				<input type='radio' bind:group={workstation} value={'All'} on:click={() => toggleWorkstation('All')} on:keydown={handleKeyPress}>
				<Icon name='all' class='all icon'/>
			</label>
			{#each workstations as ws}
				<label class='button'>
					<input type='radio' bind:group={workstation} value={ws} on:click={() => toggleWorkstation(ws)} on:keydown={handleKeyPress}>
					<span class='ws'>{ws}</span>
				</label>
			{/each}
		</div>
	{/if}
	{#if helpActive}
		<Help/>
	{/if}
</main>
<svelte:window on:keydown|preventDefault={handleKeyPress}/>