## IFC web viewer focused on custom property sets, using [IFC.js](https://ifcjs.github.io/info/) and [Svelte](https://svelte.dev/).

<img src="banner.png">

## Description
Simple IFC viewer with the ability to select the parts which need to be assembled, per workstation. As this is set in a custom property set, this can be adjusted to filter on a different custom property.

<br>

## Getting Started

### Dependencies

This app is built using `web-ifc-viewer`, which depends on `web-ifc-three`, `web-ifc`, and `three`.

### Install

Clone this repository and run `npm i`. During development `npm run dev` will serve the web application locally, if available on `http://localhost:5000`. For production, `npm run build` will serve all the files in the `public` folder. These files can then be uploaded to a server, as Svelte compiles to vanilla js.

### Contents

- `src/App.svelte`: IFC viewer and the logic behind it
- `public/global.css`: Styling of the viewer

### Functionalities and future implementations

- [x] IFC viewer: geometry and data
- [x] Creating subsets based on custom property
- [x] Isolating an element
- [x] Hiding one/multiple element(s), even on subsets
- [x] 3D clipping planes
- [x] Showing transparent elements of previous workstations
- [x] Custom Property menu: only gives select, necessary information from a custom pset

- [ ] Considering adding a Tree view
- [ ] Ability to select path for custom properties
- [ ] Adding mapbox integration
- [ ] Changing styling

## Authors

Stijn Nijhuis
[@Nomeon](https://www.linkedin.com/in/stijn-nijhuis-56593524a/)

## Version History

- 0.1
  - Initial Release

## Acknowledgments

Inspiration, code snippets, etc.

- [IFC.js](https://ifcjs.github.io/info/)
- [Francesca's viewer](https://github.com/duffra/BIMexp_o)
