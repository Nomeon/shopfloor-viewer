<script>
	import { onMount } from 'svelte';
	import { fly } from 'svelte/transition';
	import { cubicInOut } from 'svelte/easing';
	import { Color, MeshLambertMaterial } from "three";
	import { IfcViewerAPI } from "web-ifc-viewer";
	import MdFileUpload from 'svelte-icons/md/MdFileUpload.svelte';
	import MdFilter1 from 'svelte-icons/md/MdFilter1.svelte';
	import MdCrop from 'svelte-icons/md/MdCrop.svelte';
	import MdInfo from 'svelte-icons/md/MdInfo.svelte';
	import MdHome from 'svelte-icons/md/MdHome.svelte';

	// Creates subset material
	const preselectMat = new MeshLambertMaterial({
		transparent: true,
		opacity: 0.6,
		color: 0xff88ff,
		depthTest: false,
	});

	let fileInput;
	let container;
	let viewer;
	let propertyData;

	let isolateActive = false;
	let clipperActive = false;
	let detailsActive = false;
	let greyButtons = true;

	let wsObject;
	let activeModel;
	let workstations;
	let workstation = 'All';

	let selectedSubset;
	let selectedIDs = [];
	let subsets = {};


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
		// await viewer.shadowDropper.renderShadow(model.modelID);

		let JSONblob = await viewer.IFC.properties.serializeAllProperties(model);
		let JSONdata = await JSONblob[0].text();
		JSONdata = JSON.parse(JSONdata)
		const results = await createWsArray(JSONdata, model);
		[workstations, wsObject] = results;

		createSubsets(viewer, model, wsObject)
		replaceModelBySubset(viewer, model, "All")

		propertyData = await getItemProperties(model.modelID)
		greyButtons = false
	}

	function handleKeyPress(e) {
	//------------------- Handles key presses -------------------
		if (e.key == 'Escape' && viewer.clipper.active) {
			viewer.clipper.deleteAllPlanes();
		}
	}

	async function prePickItem() {
	//------------------- Pre-picks item -------------------
		await viewer.IFC.selector.prePickIfcItem();
	}

	async function pickItem() {
	//------------------- Handles item picking -------------------
		if (viewer) {
			const found = await viewer.IFC.selector.pickIfcItem();
			if (found) {
				propertyData = await getItemProperties(found.id)
				if (isolateActive) {
					selectedIDs = [];
					selectedIDs.push(found.id)
					isolate(selectedIDs)
				}
			} else {
				viewer.IFC.selector.unpickIfcItems();
			}
		}
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
	}

	function toggleClipper() {
	//------------------- Plane Button -------------------
		clipperActive = !clipperActive;
		viewer.clipper.active = clipperActive
	}

	function toggleDetails() {
	//------------------- Details Button -------------------
		detailsActive = !detailsActive;
	}

	function toggleWorkstation(ws) {
	//------------------- Workstation Button -------------------
		viewer.IFC.selector.unpickIfcItems();
		if (selectedSubset) {
			viewer.IFC.loader.ifcManager.clearSubset(activeModel.modelID, 'selected');
		}
		replaceModelBySubset(viewer, activeModel, ws)
	}

	async function createWsArray(JSONdata, model) {
	//------------------- Creates arrays of workstations and IDs -------------------
		let elementProp = [];
		let workStations = [];

		for (var id in JSONdata) {
			if (JSONdata[id]['type'] === 'IFCRELDEFINESBYPROPERTIES') {
				let elementID = JSONdata[id]['RelatedObjects'][0]
				let propSetID = JSONdata[id]['RelatingPropertyDefinition']
				elementProp.push({[elementID] : propSetID})
			}
		}

		const allIDs = Array.from(new Set(model.geometry.attributes.expressID.array),);
		let objectWS = {'All': allIDs}

		for (var key in elementProp) {
			let obj = Object.keys(elementProp[key])[0]
			let propSet = JSONdata[elementProp[key][obj]]
			for (var Sprops in propSet['HasProperties']) {
				if (JSONdata[propSet['HasProperties'][Sprops]]['Name'] === 'Station') {
					let propSetValue = JSONdata[propSet['HasProperties'][Sprops]]['NominalValue']
					if (objectWS.hasOwnProperty(propSetValue)) {
						objectWS[propSetValue].push(parseInt(obj))
					} else {
						objectWS[propSetValue] = [parseInt(obj)]
					}
					if (!workStations.includes(propSetValue)) {
						workStations.push(propSetValue)
					}
				}
			}
		}
		return [workStations, objectWS]
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
				material: preselectMat, // REMOVING BREAKS FUNCTIONALITY
			});
			subsets[key] = subset;
		}
	}

	function replaceModelBySubset(viewer, model, id) {
	//------------------- Replaces current model with subset -------------------
		const items = viewer.context.items;
		const subset = subsets[id];

		for (const key in subsets) {
			if (key !== id) {
				viewer.IFC.loader.ifcManager.removeFromSubset(model.modelID, wsObject[key], key)
				subsets[key].removeFromParent()
			}
		}
		model.removeFromParent();
		items.ifcModels = [];
		items.ifcModels.push(subset);
		activeModel = items.ifcModels[0];
		togglePickable(subset, true)
		viewer.context.scene.add(subset);
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
		if (pickable) {
			items.pickableIfcModels = [];
			items.pickableIfcModels.push(subset);
		} else {
			items.pickableIfcModels = activeModel;
		}
	}

	async function getItemProperties(expID) {
	//------------------- Gets prop data of ID -------------------
		let modID = activeModel.modelID
		const props = await viewer.IFC.getProperties(modID, expID, true)
		const data = await getPropertyGroups(props)
		return data
	}

	async function getPropertyGroups(values) {
	//------------------- Gets S parameters of item -------------------
		const props = await values
		let modID = activeModel.modelID
		const properties = await viewer.IFC.getProperties(modID, props.expressID, true, true)
		
		if (props.psets.length === 0) {
			return {name: getProp(props.LongName), description: props.constructor.name, props: [{ name: 'Name', value: getProp(props.LongName)}, { name: 'GlobalID', value: getProp(props.GlobalId)}]}
		} else {
		const propObj = await properties['psets'][0]['HasProperties'].map((p) => {
			return { name: getProp(p.Name), value: getProp(p.NominalValue) }
		})
		try {
			propObj.splice(16, 1);
			propObj.splice(9, 6);
		} catch(e) {}
			return {name: props.Name.value, description: props.constructor.name, props: propObj}
		}
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
	<aside class="toolbar">
		<input style="display:none" type="file" accept=".ifc" on:change={(e)=>loadIfc(e)} bind:this={fileInput} >
		<button class='button' on:click={()=>{fileInput.click()}} on:keydown={handleKeyPress}>
			<div class='icon'>
				<MdFileUpload/>
			</div>
			<span class='tooltip'>Upload een IFC bestand</span>
		</button>
		<button class='button' class:selected="{isolateActive}" class:non-active="{greyButtons}" on:click={toggleIsolate(workstation)} on:keydown={handleKeyPress}>
			<div class='icon'>
				<MdFilter1/>
			</div>
			<span class='tooltip'>Isoleer een element</span>
		</button>
		<button class='button' class:selected="{clipperActive}" class:non-active="{greyButtons}" on:click={toggleClipper} on:keydown={handleKeyPress}>
			<div class='icon'>
				<MdCrop/>
			</div>
			<span class='tooltip'>Maak doorsnedes</span>
		</button>
		<button class='button' class:selected="{detailsActive}" class:non-active="{greyButtons}" on:click={toggleDetails} on:keydown={handleKeyPress}>
			<div class='icon'>
				<MdInfo/>
			</div>
			<span class='tooltip'>Geef properties weer</span>
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
							<th id='col1'>Property</th>
							<th id='col2'>Value</th>
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
				<div class='all icon'>
					<MdHome/>
				</div>
			</label>
			{#each workstations as ws}
				<label class='button'>
					<input type='radio' bind:group={workstation} value={ws} on:click={() => toggleWorkstation(ws)} on:keydown={handleKeyPress}>
					<span class='ws'>{ws}</span>
				</label>
			{/each}
		</div>
	{/if}
	<div id="viewer-container"
		bind:this={container}
		on:click={pickItem}
		on:keydown={handleKeyPress}
		on:mousemove={prePickItem}
		on:dblclick={() => {if (viewer.clipper.active) {viewer.clipper.createPlane()}}}>
	</div>
</main>
<svelte:window on:keydown|preventDefault={handleKeyPress}/>