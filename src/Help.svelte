<script>
    import Icon from './Icon.svelte';
</script>

<div class='help-container'>
    <div class='help-heading'>
        <h1>Gebruiksaanwijzing IFC-viewer</h1>
        <h4>Klik op de knop met de gewenste functionaliteit om deze te activeren. Hiervoor kan ook het toetsenbord gebruikt worden; de eerste letter (in het <span>groen</span>) geeft aan welke toets daarvoor nodig is. Klik daarna op een element om de bewerking toe te passen.</h4>
    </div>
    <div class='help-table'>
        <div class='help-row'>
            <Icon name='upload' class='icon help-icon'/>
            <div class='button-explanation'>Met de upload knop kan een IFC ingeladen worden. Deze wordt uitgeschakeld nadat er een is ingelaten. Mocht er een nieuwe IFC nodig zijn, hoeft alleen de pagina ververst te worden.</div>
        </div>
        <div class='help-row'>
            <Icon name='hide' class='icon help-icon'/>
            <div class='button-explanation'>De <span>V</span>erberg knop kan een of meer elementen verbergen, zodat de onderliggende elementen zichtbaar worden.</div>
        </div>
        <div class='help-row'>
            <Icon name='isolate' class='icon help-icon'/>
            <div class='button-explanation'>De <span>I</span>soleer knop verbergt alle elementen, behalve het geselecteerde element.</div>
        </div>
        <div class='help-row'>
            <Icon name='section' class='icon help-icon'/>
            <div class='button-explanation'>Om <span>D</span>oorsnedes te maken, activeer deze knop, en dubbelklik op een element. Doorsnedes kunnen tijdelijk worden uitgezet door de knop te deactiveren, en verwijderd worden met Escape.</div>
        </div>
        <div class='help-row'>
            <Icon name='measure' class='icon help-icon'/>
            <div class='button-explanation'>De <span>M</span>eet tool berekent de afstand tussen 2 punten. Dubbelklik op het begin en het eindpunt, en het label geeft de afstand in meters weer. Labels kunnen tijdelijk worden uitgezet door de knop te deactiveren, en verwijderd worden met Escape.</div>
        </div>
        <div class='help-row'>
            <Icon name='transparent' class='icon help-icon'/>
            <div class='button-explanation'>Met de <span>T</span>ransparante lagen knop, worden de elementen die zijn toegevoegd in de vorige werkstations weergegeven.</div>
        </div>
        <div class='help-row'>
            <Icon name='details' class='icon help-icon'/>
            <div class='button-explanation'>De <span>D</span>etails knop geeft de eigenschappen (properties) van het geselecteerde element weer.</div>
        </div>
        <div class='help-row'>
            <Icon name='all' class='icon help-icon'/>
            <div class='button-explanation'>De knoppen in deze rij geven de elementen in de desbetreffende werkstations weer.</div>
        </div>
    </div>

</div>