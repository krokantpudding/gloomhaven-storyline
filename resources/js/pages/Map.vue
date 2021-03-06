<template>
    <div id="map-container" class="h-screen w-screen overflow-hidden">
        <webp src="/img/map-background-lowres.jpg" :cover="true"
              highres="/img/map-background-highres.jpg"
              alt="Gloomhaven map background" class="fixed"/>
        <div id="map">
            <webp src="/img/map-lowres.jpg"
                  highres="/img/map-highres.jpg"
                  alt="Gloomhaven map"
                  class="map"/>
            <ul v-if="achievements">
                <li v-for="achievement in achievements.items"
                    v-if="achievement.isGlobal() && achievement.awarded && !achievement.hidden"
                    :key="achievement.id"
                    class="absolute"
                    @click="openAchievement(achievement)"
                    :style="{
                        'top': '-174px',
                        'left': achievement.x + '%'
                    }">
                    <webp :src="achievement.image"
                          :animate="true" :retina="true"
                          class="cursor-pointer"
                          :alt="achievement.displayName"/>
                </li>
            </ul>
            <template v-if="scenarios">
                <webp v-for="scenario in scenarios.items"
                      v-if="scenario.isVisible()"
                      :src="scenario.image()"
                      :key="scenario.id"
                      :id="'s' + scenario.id"
                      :animate="true"
                      :alt="scenario.name"
                      class="absolute scenario"
                      :style="{
                          'left': scenario.coordinates.x + '%',
                          'top': scenario.coordinates.y + '%',
                      }"/>
            </template>
        </div>
    </div>
</template>

<script>
    import panzoom from "panzoom";
    import Hammer from 'hammerjs';
    import tippy from "tippy.js";

    export default {
        data() {
            return {
                map: null,
                $map: null,
                mapTouch: null,
                scenarios: null,
                achievements: null
            }
        },
        mounted() {
            this.$map = $('#map');
            this.map = panzoom(this.$map[0], {
                minZoom: this.scale(),
                maxZoom: 4,
                bounds: true
            });
            this.centerMap();

            if (app.scenarios) {
                this.setScenarios();
            }

            this.$bus.$on('scenarios-updated', this.setScenarios);
            this.$bus.$on('windows-resized', this.setScenarios);
            $('#map').on('click', '.scenario', this.scenarioClicked);
            this.mapTouch = new Hammer(this.$map[0]);
            this.mapTouch.on('tap', (e) => {
                if (e.target.id.startsWith('s')) {
                    this.scenarioClicked(e);
                }
            });

            if (app.achievements) {
                this.setAchievements();
            }
            this.$bus.$on('achievements-updated', this.setAchievements);
        },
        destroyed() {
            if (this.map) {
                this.map.dispose();
            }
            this.$bus.$off('scenarios-updated', this.setScenarios);
            this.$bus.$off('windows-resized', this.setScenarios);
            $('#map').off('click', '.scenario', this.scenarioClicked);
            this.mapTouch.destroy();
        },
        methods: {
            setScenarios() {
                this.map.setMinZoom(this.scale());

                this.scenarios = app.scenarios;

                // Show tooltip on hover
                this.scenarios.each((scenario) => {
                    let $s = $('#s' + scenario.id);
                    if ($s.length && app.hasMouse && scenario.isVisible() && !$s.hasClass('tippy')) {
                        tippy($s[0], {
                            content: scenario.title
                        });
                        $s.addClass('tippy');
                    }
                });
            },
            setAchievements() {
                this.achievements = app.achievements;
            },
            scenarioClicked(e) {
                let id = parseInt(e.target.id.replace('s', ''));
                this.open(id);
            },
            open(id) {
                this.$bus.$emit('open-scenario', {
                    id: id
                });
            },
            openAchievement(achievement) {
                this.$bus.$emit('open-achievement', {
                    id: achievement.id
                });
            },
            centerMap() {
                const scale = this.scale();
                this.map.zoomTo(0, 0, scale);
                const yOffset = 184 * scale;

                if (this.isLandscape()) {
                    const mapWidth = this.$map.width() * scale;
                    let x = ($(window).width() - mapWidth) / 2;
                    this.map.moveTo(x, yOffset + (20 / scale));
                } else {
                    const mapHeight = this.$map.height() * scale;
                    let y = ($(window).height() - mapHeight) / 2;
                    this.map.moveTo(0, y + yOffset);
                }
            },
            isLandscape() {
                return $(window).width() > $(window).height();
            },
            scale() {
                return this.isLandscape()
                    ? $(window).height() / 2296
                    : $(window).width() / this.$map.width();
            }
        }
    }
</script>

<style lang="scss">
    #map {
        width: 2484px;
        height: 2160px;
        position: relative;

        .map {
            top: -184px;
            left: 0;
            width: 100%;
            position: absolute;
        }

        .scenario {
            transform: scale(0.79);

            &:hover {
                cursor: pointer;
            }
        }
    }
</style>
