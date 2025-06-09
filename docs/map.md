<iframe id="pokemonMap" width="100%" allow="geolocation 'self' https://server.nicbk.com" src="https://server.nicbk.com/#/map"></iframe>
<script>
    'use strict';
    (() => {
        const map = document.getElementById('pokemonMap');

        const resizeMap = (_event) => {
            map.style.height = `${parseInt(0.8 * window.innerHeight)}px`;
        };

        resizeMap();

        window.addEventListener('resize', resizeMap);
    })();
</script>