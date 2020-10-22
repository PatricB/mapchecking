<template>
    <div class="w-full h-full">
<!--        <input id="pac-input" class="controls" :class="[mapLoaded ? '' : 'hidden']" type="text" placeholder="Search Box">-->
        <div id="map" class="w-full h-full"></div>
    </div>
</template>

<script>
    import Leaflet from 'leaflet';
    import area from '@turf/area'; // TODO: check why not correct loaded
    import { Base64 } from 'js-base64';
    import 'leaflet/dist/leaflet.css';

    export default {
        name: 'Map',

        props: {
            density: Number,
            startHash: String
        },

        mounted () {
            const map = Leaflet.map('map', {
                center: [this.mapPosition[0], this.mapPosition[1]],
                zoom: this.mapPosition[2]
            });
            
            // map.on('bounds_changed', function() {
            //     searchBox.setBounds(map.getBounds());
            // });
            map.on('moveend', this.mapUpdated);
            map.on('zoomend', this.mapUpdated);
            map.on('click', this.mapClicked);

            Leaflet.DomUtil.addClass(map._container,'crosshair-cursor-enabled');


            Leaflet.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png?{foo}',
                {
                    foo: 'bar',
                    attribution: 'Map data &copy; <a href="https://www.openstreetmap.org/">OpenStreetMap</a> contributors, <a href="https://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>'
                }).addTo(map);

            // const input = document.getElementById('pac-input');

            this.$map = map;            
            
            this.polygon = Leaflet.polygon([], {
                fillOpacity: 0.35,
                fillColor: `hsl(0, 90%, 50%)`,
                weight: 2,
                opacity: 0.8,
                color: `hsl(0, 90%, 50%)`,
                smoothFactor: 1.5,
                noClip: true
            }).addTo(map)

            if (this.startHash) {
                this.loadHash(this.startHash);
            }
            
            this.updatePolygonColor();

            this.$updateHashTimer = null;
            this.mapLoaded = true;
        },

        watch: {
            density (val) {
                this.updatePolygonColor();
            },

            hash (hashval) {
                if (this.$updateHashTimer) {
                    clearTimeout(this.$updateHashTimer);
                }

                this.$updateHashTimer = setTimeout(() => {
                    this.$emit('hashChange', hashval);
                }, 300);
            },

            polygonPath: {
                handler (value) {
                    this.polygon.setLatLngs(value)
                    this.surfaceUpdated()
                },
                deep: true
            }
        },

        methods: {
            
            getHue (val) {
                const min = 0.1;
                const max = 3.0;

                // inv lerp from the density
                const t = (val - min) / (max - min);

                // lerp between green and red hue
                const hue = (1.0 - t) * 110 + 0 * t;

                return Math.max(0, Math.min(hue, 110));
            },

            updatePolygonColor () {
                const hue = this.getHue(this.density);

                this.polygon.setStyle({
                    fillColor: `hsl(${hue}, 90%, 50%)`,
                    strokeColor: `hsl(${hue}, 90%, 50%)`
                })
            },

            mapUpdated () {
                const pos = this.$map.getCenter();
                const zoom = this.$map.getZoom();

                this.mapPosition = [pos.lat.toFixed(7), pos.lng.toFixed(7), zoom];
            },

            mapClicked (ev) {
                this.polygonPath.push([ev.latlng.lat, ev.latlng.lng])
            },

            surfaceUpdated () {
                this.surface = area.default(this.polygon.toGeoJSON()).toFixed(2)
                this.arrPoly = this.polygonPath.slice()
                this.$emit('surfaceUpdate', this.surface);
            },

            reloadHash (hash) {
                this.loadHash(hash);
                this.updatePolygonColor();
            },

            loadHash (hash) {
                if (hash[0] != 'b') {
                    return this.loadLegacyHash(hash);
                }

                const buf = Base64.toUint8Array(hash.substr(1));
                if (!buf) {
                    return;
                }

                const meta = new Float32Array(buf.buffer, 0, 4);
                const data = new Float32Array(buf.buffer, 4 * 4);

                this.$map.panTo([meta[1], meta[2]]);
                this.$map.setZoom(parseInt(meta[3]));

                let path = [];
                for (let i = 0; i < data.length; i += 2) {
                    path.push([data[i], data[i + 1]]);
                }
                
                if (path.length) {
                    this.polygonPath = path;
                }

                this.$emit('densityChange', parseInt(meta[0]));
            },

            loadLegacyHash (hash) {
                let opt = hash.split(';');

                let curPosition = opt.pop();

                if (curPosition) {
                    let cursetting = curPosition.split(',');
                    this.$map.panTo([parseFloat(cursetting[0]), parseFloat(cursetting[1])]);
                    this.$map.setZoom(parseInt(cursetting[2]));
                }

                let density = parseFloat(opt.pop()) || 1;

                let path = [];

                for (let i = 0; i < opt.length; i++) {
                    let coord = opt[i].split(',');
                    path.push([parseFloat(coord[0]), parseFloat(coord[1])]);
                }

                if (path.length) {
                    this.polygonPath = path;
                }

                this.$emit('densityChange', density);
            },

            reset () {
                this.polygonPath = []
            }
        },

        computed: {
            hash () {
                let buf = new Float32Array(this.arrPoly.length * 2 + 4);
                buf[0] = this.density;
                buf.set(this.mapPosition, 1);

                for (let i = 0; i < this.arrPoly.length; i++) {
                    buf[4 + i * 2] = this.arrPoly[i][0];
                    buf[4 + i * 2 + 1] = this.arrPoly[i][1];
                }
                return 'b' + Base64.fromUint8Array(new Uint8Array(buf.buffer), true);
            }
        },

        data () {
            return {
                mapPosition: [48.862895, 2.286978, 18],
                surface: 0,
                arrPoly: [],
                mapLoaded: false,
                polygon: null,
                polygonPath: [],
            };
        }
    };
</script>
