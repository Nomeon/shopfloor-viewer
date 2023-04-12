## IFC web viewer focused on custom property sets, using [IFC.js](https://ifcjs.github.io/info/) and [Svelte](https://svelte.dev/).
*Heavily inspired by [Francesca's viewer](https://github.com/duffra/BIMexp_o).*

<img src="banner.png">

<br>

## Brief explanation
By using the function
```js
viewer.IFC.properties.serializeAllProperties(model)
```
The JSON properties blob can be traversed more efficiently. 
For my use case, the property I want is the workstation property so I can filter the components per station.

<br>

By getting the [IFCRELDEFINESBYPROPERTIES](https://standards.buildingsmart.org/IFC/RELEASE/IFC2x3/TC1/HTML/ifckernel/lexical/ifcreldefinesbyproperties.htm), 
the object can be linked to it's custom property sets. 
```js
function createWsArray() {
  //------------------- Creates array of workstations and IDs -------------------
}
```
This function provides more clarity on the matter.

<br>

In addition, the following function gets the custom set I need, this function also needs to be adapted in order to function for IFCs with different property sets:
```js
async function getPropertyGroups(values) {
  //------------------- Gets S parameters of item -------------------
}
```

<br>

## Installing
Due to the dependencies issues that are present (at the time of writing this) in IFC.js/Three.js, I suggest cloning the repository and using 
``` 
npm ci 
```
