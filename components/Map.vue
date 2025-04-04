
<script setup>
    import { ref, onMounted, useAttrs, watch, getCurrentInstance, onBeforeUnmount } from 'vue'
    import {Deck, WebMercatorViewport, TRANSITION_EVENTS} from '@deck.gl/core';
    import {GeoJsonLayer, BitmapLayer, ScatterplotLayer} from '@deck.gl/layers';
    import {HexagonLayer} from '@deck.gl/aggregation-layers';
    import {TileLayer} from '@deck.gl/geo-layers';

    import {bboxPolygon, difference, featureCollection} from '@turf/turf';
    import { createClient } from '@supabase/supabase-js';

    import { useGlobalTheme } from '../.vitepress/theme/composables/useGlobalTheme'
    const { isDark } = useGlobalTheme()

    const instance = getCurrentInstance();
    const apikey = instance.appContext.config.globalProperties.$apikey;
    const url = instance.appContext.config.globalProperties.$url;
    const supabase = createClient(url, apikey)

    const attrs = useAttrs();

    const selectByClick = attrs.selectByClick;
    const parentHover = attrs.parentHover;

    const mapElement = ref(null);
    const deckInstance = ref(null);
    const bbMask = ref(null);
    let tooltip = null;
    
    const INERTIA_EASING = t => 1 - (1 - t) * (1 - t);
    const boundingBox = [
        [12.0833, 51.3411],
        [14.6596, 53.5511]
    ];// npm install @deck.gl/core@8.9.36	 @deck.gl/layers@8.9.36	 @deck.gl/geo-layers@8.9.36	

    const fitBounds = (bounds, options = {}) => {
        if (!deckInstance.value) return;
        if (!mapElement.value) return;
        // get mapElement width
        const mapViewState = mapElement.value.getBoundingClientRect();

        let {longitude, latitude, zoom} = new WebMercatorViewport({ width: mapViewState.width, height: mapViewState.width*0.9 }).fitBounds(bounds)
        zoom = Math.min(zoom, 19);

        if(typeof bounds !== 'object' ) return;
        deckInstance.value.setProps({viewState: {
            ...deckInstance.value.props.viewState,
            longitude,
            latitude,
            zoom: zoom,
            pitch: 25,
            //transitionInterpolator: new FlyToInterpolator({speed: 2000}),
            transitionDuration: 1000,
            transitionInterruption: TRANSITION_EVENTS.SNAP_TO_END,
            transitionEasing: INERTIA_EASING,
            
        }});
    };

    const INITIAL_VIEW_STATE = {
        latitude: 52.520008,
        longitude: 13.404954,
        zoom: 5,
        //pitch: 30,
        bearing: 0,
    };

    // Function to convert DDMMSS format to decimal degrees
    function convertDMSToDecimal(dmsValue) {
        // Handle negative values (Western longitudes/Southern latitudes)
        const isNegative = dmsValue < 0;
        let absDms = Math.abs(dmsValue);
        
        // Extract degrees, minutes, and seconds
        const degrees = Math.floor(absDms / 10000);
        absDms -= degrees * 10000;
        
        const minutes = Math.floor(absDms / 100);
        const seconds = absDms - minutes * 100;
        
        // Calculate decimal degrees
        let decimalValue = degrees + (minutes / 60) + (seconds / 3600);
        
        // Apply negative sign if needed
        return isNegative ? -decimalValue : decimalValue;
    }

    const polyMask = (isochrone) => {
        const bboxPoly = bboxPolygon([-180, -85, 180, 85]);
        const mask = difference(featureCollection([bboxPoly, isochrone]));
        return mask;
    }
    const loadData = async (table, uniquenessColumn, latitudeColumn, longitudeColumn, dataColumn) => {
        const addedFeatures = [];
        
        try {
            // Use await without .then() chain
            const response = await supabase.schema('icp_download').from(table).select(`*`);
            
            if (response.error) {
                console.error('Error loading data:', response.error);
                return { data: null, error: response.error };
            }
            
            const data = response.data;
            const features = [];
            
            for (let i = 0; i < data.length; i++) {
                const item = data[i];
                // convert +/-DDMMSS in Degrees Minutes Seconds (WGS84) to decimal degrees
                const lat = convertDMSToDecimal(item[latitudeColumn]);
                const lon = convertDMSToDecimal(item[longitudeColumn]);
                
                // check if the item is already added
                if (addedFeatures.includes(item[uniquenessColumn])) {
                    continue;
                }
                
                // add the item to the addedFeatures array
                addedFeatures.push(item[uniquenessColumn]);
                features.push({
                    type: 'Feature',
                    geometry: {
                        type: 'Point',
                        coordinates: [lon, lat]
                    },
                    properties: {
                        ...item,
                        highlighted: item[dataColumn],
                    }
                });
            }
            
            const layer = new ScatterplotLayer({
                id: 'data-scatterplot-layer',
                data: features,
                getPosition: d => d.geometry.coordinates,
                getRadius: d => Math.sqrt(d.properties.value) * 100,
                getFillColor: [70, 119, 76],
                pickable: true,
                radiusMinPixels: 5,
                radiusMaxPixels: 15,
            });
            
            const layers = deckInstance.value.props.layers;
            layers[2] = layer;
            deckInstance.value.setProps({
                layers: [...layers]
            });
            
            return { data: features, error: null };
        } catch (error) {
            console.error('Error loading data:', error);
            return { data: null, error };
        }
    }
    const updateDataLayer = async (data) => {
        const layers = deckInstance.value.props.layers;
        layers[2] = new HexagonLayer({
            id: 'HexagonLayer',
            data,
            gpuAggregation: true,
            extruded: true,
            getPosition: d => {
                return [d.coordinates[0], d.coordinates[1]];
            },
            getColorWeight: d => d.daily_mean/10 || 1,
            getElevationWeight: d => d.daily_mean || 1,
            colorRange: [
                [1, 152, 189, 150],
                [73, 227, 206, 150],
                [216, 254, 181, 150],
                [254, 237, 177, 150],
                [254, 173, 84, 150],
                [209, 55, 78, 150]
            ],
            elevationScale: 10,
            radius: 5000,
            pickable: true,
            transitions: {
                getElevationWeight: 300
            }
        });

        deckInstance.value.setProps({
            layers: [...layers]
        });
    }
    const _loadMask = async () => {
        await fetch('/brandenburg.geojson')
            .then(response => response.json())
            .then(data => {
                bbMask.value = data.features[0];
                //const mask = polyMask(data);
                //bbMask.value = mask;
            })
            .catch(error => {
                console.error('Error loading GeoJSON:', error);
            });
        const layers = deckInstance.value.props.layers;

        layers[1] = new GeoJsonLayer({
                    id: 'mask-layer-0_8',
                    data: polyMask(bbMask.value),
                    getFillColor: isDark.value ? [27,27,31,255] : [255, 255, 255, 255],
                    stroked: false,
                    extruded: false,
                    wireframe: true,
                    lineJointRounded: true,
                });
        deckInstance.value.setProps({
            layers: [...layers]
        });
    }

    
    

    function getTooltip({object, x, y}) {
        if(!object) {
            tooltip.style.display = 'none';
            return;
        }
        tooltip.style.display = 'block';
        tooltip.style.left = `${x}px`;
        tooltip.style.top = `${y}px`;
        if (parentHover && object?.properties) {
            parentHover({object, x, y}, tooltip);
        }else if(object?.properties?.highlighted) {
            tooltip.innerText = object?.properties.highlighted;
        }
    }

    
    let onClick = (info, event) => {
        if(selectByClick) {
            selectByClick(info, event);
        }
    };
    
    onMounted(async () => {
        tooltip = document.createElement('div');
        tooltip.style.position = 'absolute';
        tooltip.style.zIndex = 1;
        tooltip.style.pointerEvents = 'none';
        tooltip.style.backgroundColor = isDark.value ? 'rgba(27,27,31,0.8)' : 'rgba(255, 255, 255, 0.8)';
        tooltip.style.color = isDark.value ? 'white' : 'black';
        tooltip.style.padding = '5px';
        tooltip.style.borderRadius = '5px';
        tooltip.style.display = 'none';
        mapElement.value.appendChild(tooltip);

        deckInstance.value = new Deck({
            mapStyle: 'https://basemaps.cartocdn.com/gl/positron-nolabels-gl-style/style.json',
            parent: mapElement.value,
            style: {
                position: 'relative',
                aspectRatio: 1,
                width: '100%',
            },
            initialViewState: INITIAL_VIEW_STATE,
            controller: false,
            onResize: () => {
                setTimeout(() => {
                    fitBounds(boundingBox);
                }, 100);
                
            },
            onClick,
            getTooltip,
            layers: [
                new TileLayer({
                    id: 'tile-layer',
                    data: [
                        'https://a.tile.openstreetmap.org/{z}/{x}/{y}.png',
                        'https://b.tile.openstreetmap.org/{z}/{x}/{y}.png',
                        'https://c.tile.openstreetmap.org/{z}/{x}/{y}.png'
                    ],
                    pickable:true,
                    renderSubLayers: props => {
                        const {
                            bbox: {west, south, east, north}
                        } = props.tile;
                
                        return [
                            new BitmapLayer(props, {
                                data: null,
                                image: props.data,
                                bounds: [west, south, east, north]
                            })
                        ];
                    }
                })
            ]
        });
        await _loadMask();
    });
    defineExpose({
        deckInstance,
        fitBounds,
        loadData,
        updateDataLayer
    });
</script>


<template>
    <div  style="position:relative">
    <div ref="mapElement"></div>
    </div>
</template>
