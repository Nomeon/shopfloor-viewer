<script>
	// Transparent mesh creation
	// Next station -> previous transparent (other meshes transparent)
	// BUG: Hiding element on WS
	// BUG: Isolating element resets to full model
	// View

	import { onMount } from 'svelte';
	import { Color } from "three";
	import { IfcViewerAPI } from "web-ifc-viewer";
	import { MeshLambertMaterial } from "three";
	import MdFileUpload from 'svelte-icons/md/MdFileUpload.svelte';
	import MdInfo from 'svelte-icons/md/MdInfo.svelte'
	import MdHome from 'svelte-icons/md/MdHome.svelte'
	import MdVisibilityOff from 'svelte-icons/md/MdVisibilityOff.svelte'
	import MdFilter1 from 'svelte-icons/md/MdFilter1.svelte'

	// Creates subset material
	const preselectMat = new MeshLambertMaterial({
		transparent: true,
		opacity: 0.6,
		color: 0xff88ff,
		depthTest: false,
	});

	let fileInput;
	let container;
	let propertyData;

	let isolateActive = false;
	let hideActive = false;
	let detailsActive = false;
	let greyButtons = true;

	let viewer;
	let activeModel;
	let workstations;
	let workstation = 'All';
	let wsObject;

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
      		USE_FAST_BOOLS: true
    	})
		viewer.IFC.setWasmPath("wasm/");
		return viewer;
	}

	const loadIfc = async (event) => {
	//------------------- Loading IFC -------------------
		const model = await viewer.IFC.loadIfc(event.target.files[0]);
		await viewer.shadowDropper.renderShadow(model.modelID);

		let JSONblob = await viewer.IFC.properties.serializeAllProperties(model);
		let JSONdata = await JSONblob[0].text();
		JSONdata = JSON.parse(JSONdata)
		const results = await createWsArray(JSONdata, model);
		[workstations, wsObject] = results;

		setupEvents(viewer, model);
		createSubsets(viewer, model, wsObject)
		replaceModelBySubset(viewer, model, "All")

		propertyData = await getItemProperties(model.modelID)
		greyButtons = false
	}

	function toggleDetails() {
	//------------------- Details Button -------------------
		detailsActive = !detailsActive;
		const scene = viewer.context.getScene();
		console.log(scene)
	}

	function setupEvents(viewer, model) {
	//------------------- Sets up events -------------------
		window.onmousemove = async () => {
			await viewer.IFC.selector.prePickIfcItem();
		}
		window.onclick = async () => {
			viewer.IFC.selector.unpickIfcItems();
			const selected = await viewer.IFC.selector.pickIfcItem();
			if (!selected) return;

			if (isolateActive) {
				let selectedIDs = [];
				selectedIDs.push(selected.id);
				isolate(selectedIDs, viewer, model);
			};

			if (hideActive) {
				selectedIDs.push(selected.id);
				hide(selectedIDs, viewer, model);
			};
		};
	}

	function toggleWorkstation(ws) {
	//------------------- Workstation Button -------------------
		replaceModelBySubset(viewer, activeModel, ws)
	}

	async function createWsArray(JSONdata, model) {
	//------------------- Creates array of workstations and IDs -------------------
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
				removePrevious: false,
				customID: key,
				material: preselectMat, // REMOVING BREAKS FUNCTIONALITY
			});
			subsets[key] = subset;
		}
	}

	function replaceModelBySubset(viewer, ifcModel, id) {
		const items = viewer.context.items;
		const subset = subsets[id];
		items.pickableIfcModels = items.pickableIfcModels.filter((model) => model.modelID !== ifcModel.modelID);

		for (const key in subsets) {
			if (key !== id) {
				viewer.IFC.loader.ifcManager.removeFromSubset(0, wsObject[key], key)
				subsets[key].removeFromParent()
			}
		}
		ifcModel.removeFromParent();
		items.ifcModels = [];
		items.ifcModels.push(subset);
		activeModel = subset;
		items.pickableIfcModels.push(subset);
		viewer.context.scene.add(subset);
	}


	async function setPropExpressId() {
	//------------------- Handles item picking -------------------
		if (viewer) {	// && activeMod === 'All'
			const found = await viewer.IFC.selector.pickIfcItem();
			if (found) {
				propertyData = await getItemProperties(found.id)
			} else {
				viewer.IFC.selector.unpickIfcItems();
			}
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
		<!-- svelte-ignore a11y-click-events-have-key-events -->
		<button class='button' on:click={()=>{fileInput.click();}}>
			<div class='icon'>
				<MdFileUpload/>
			</div>
			<span class='tooltip'>Upload een IFC bestand</span>
		</button>
		<!-- svelte-ignore a11y-click-events-have-key-events -->
		<button class='button' on:click={toggleDetails} class:selected="{detailsActive}" class:non-active="{greyButtons}">
			<div class='icon'>
				<MdInfo/>
			</div>
			<span class='tooltip'>Geef properties weer</span>
		</button>
	</aside>
	{#if detailsActive}
	<div class="side-menu-right">
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
		<div class='wsbar'>
			<!-- svelte-ignore a11y-click-events-have-key-events -->
			<label class='button'>
				<input type='radio' bind:group={workstation} value={'All'} on:click={() => toggleWorkstation('All')}>
				<div class='all icon'>
					<MdHome/>
				</div>
			</label>
			{#each workstations as ws}
				<!-- svelte-ignore a11y-click-events-have-key-events -->
				<label class='button'>
					<input type='radio' bind:group={workstation} value={ws} on:click={() => toggleWorkstation(ws)}>
					<span class='ws'>{ws}</span>
				</label>
			{/each}
		</div>
	{/if}
	<!-- svelte-ignore a11y-click-events-have-key-events -->
	<div id="viewer-container"
		bind:this={container}
		on:click={setPropExpressId}>
	</div>
</main>