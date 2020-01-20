<template>
    <div class="trendline" v-if="loaded">
        <vue-apex-charts ref="chart" :type="type" :options="n_options" :series="n_series"/>
    </div>
</template>

<script>
    import VueApexCharts from "vue-apexcharts";

    const merge = require('deepmerge');

    const defaultTrendOptions = () => {
        return {
            text: 'Trend Line',
            indices: undefined,
            series: undefined,
            show: true,
            tooltips: false,
            dataLabels: false,
            combined: undefined,
        };
    };

    const defaultRegression = () => {
        return {
            sums: {
                x: 0,
                y: 0,
                xy: 0,
                x2: 0,
                y2: 0,
                count: 0,
            },
            a: 0,
            b: 0,
        };
    };

    const defaultYMax = () => {
        return 6;
    };

    export default {
        name: "VueApexchartsTrendline",
        components: {
            VueApexCharts,
        },
        props: {
            type: {
                // chart type for the main series to be displayed
                type: String,
                default: 'area',
            },
            options: {
                // options for ApexCharts
                type: Object,
                default: () => {
                    return {
                        title: undefined, // added property to prevent deepmerge from throwing error.
                    };
                },
            },
            series: {
                // each set requires a "data" array with "x" and "y" properties.
                // eg. [{x: 12, y: 23}, ...]
                type: Array,
                required: true,
            },
            trendOptions: {
                type: Object,
                default: defaultTrendOptions,
            },
        },
        data() {
            return {
                _series: [],
                regression: {},
                y_max: defaultYMax(),
                loaded: false,
            };
        },
        computed: {
            n_options() {
                let options = {},
                    i;

                // check if trend line is to be shown
                if (this.showTrendLine) {
                    // set yaxis max value and formatter to prevent too many decimal places showing
                    let max = this.getMaxY() * 1.1;
                    options.yaxis = {
                        max: max,
                        labels: {
                            formatter(value) {
                                return Math.round(value);
                            },
                        },
                    };

                    if (! this.t_options.tooltips) {
                        // remove tooltips for trend lines
                        options.tooltip = {
                            enabledOnSeries: [],
                        };

                        let inverse = false;

                        try {
                            inverse = this.options.tooltip.inverseOrder;
                        } catch (e) {}

                        for (i = 0; i < this.series.length; i++) {
                            if (inverse) {
                                options.tooltip.enabledOnSeries.push(this.n_series.length - (i + 1));
                            } else  {
                                options.tooltip.enabledOnSeries.push(i);
                            }
                        }
                    }

                    if (! this.t_options.dataLabels) {
                        // remove dataLabels for trend lines
                        options.dataLabels = {
                            enabledOnSeries: [],
                        };

                        for (i = 0; i < this.series.length; i++) {
                            options.dataLabels.enabledOnSeries.push(i);
                        }
                    }
                }

                return merge(this.options, options);
            },
            n_series() {
                let series = [],
                    set,
                    i;

                for (i = 0; i < this.series.length; i++) {
                    set = this.series[i];
                    series.push(set);
                }

                if (this.showTrendLine && this._series && this._series.length > 0) {
                    for (i = 0; i < this._series.length; i++) {
                        set = this._series[i];
                        series.push(set);
                    }
                }

                return series;
            },
            t_options() {
                return {...defaultTrendOptions(), ...this.trendOptions};
            },
            showTrendLine() {
                return !! this.t_options.show;
            },
            combinedTrendLines() {
                if (this.t_options.combined !== undefined) {
                    return this.t_options.combined;
                }

                let stacked = false;

                try {
                    stacked = this.options.chart.stacked;
                } catch (e) {}

                return stacked;
            },
            trendLineSeries() {
                let series,
                    data = [],
                    i;

                if (this.t_options.series !== undefined) {
                    series = this.t_options.series;
                } else {
                    series = this.series;
                }

                if (this.t_options.indices !== undefined) {
                    for (i = 0; i < this.t_options.indices.length; i++) {
                        data.push(series[this.t_options.indices[i]]);
                    }
                } else {
                    data = series;
                }

                return data;
            },
            trendText() {
                return this.t_options.text;
            },
        },
        watch: {
            trendLineSeries() {
                if (this.showTrendLine) {
                    this.setTrendLine();
                }
            },
        },
        methods: {
            setTrendLine() {
                this._series = [];
                this.y_max = defaultYMax();

                if (this.combinedTrendLines) {
                    this.processCombinedSeries(this.trendLineSeries);
                } else {
                    this.processSeries(this.trendLineSeries);
                }
            },
            processSeries(series) {
                series.forEach(this.appendToSeries);
            },
            appendToSeries(item) {
                this._series.push({
                    name: item.name + ' : ' + this.trendText,
                    data: this.processData(item.data),
                    type: 'line',
                    stacked: false,
                });
            },
            processCombinedSeries(series) {
                let data = this.combineSeries(series);

                this._series.push({
                    name: this.trendText,
                    data: this.processData(data),
                    type: 'line',
                    stacked: false,
                });
            },
            combineSeries(series) {
                let data = this.cloneObject(series[0]),
                    set,
                    i,
                    item,
                    j;

                if (typeof data.data === 'undefined') {
                    return [];
                }

                data = data.data;

                for (i = 1; i < series.length; i++) {
                    set = series[i];

                    for (j = 0; j < set.data.length; j++) {
                        item = set.data[j];
                        data[j].y += item.y;
                    }
                }

                return data;
            },
            processData(data) {
                let processed = [],
                    item,
                    i;

                if (data.length === 0) {
                    return processed;
                }

                // set the regression values to default
                this.regression = defaultRegression();

                // calculate the regression new sums
                for (i = 0; i < data.length; i++) {
                    this.addToRegressionSums(data[i]);
                }

                // calculate "a" and "b" values for the regression equation.
                this.regression.a = this.calculateAValue();
                this.regression.b = this.calculateBValue();

                // get "y" values for the trend line.
                for (i = 0; i < data.length; i++) {
                    item = data[i];

                    processed.push({
                        x: item.x,
                        y: this.calculateYValue(item.x),
                    });
                }

                return processed;
            },
            addToRegressionSums(item) {
                this.regression.sums.x += item.x;
                this.regression.sums.y += item.y;
                this.regression.sums.xy += (item.x * item.y);
                this.regression.sums.x2 += (item.x ** 2);
                this.regression.sums.y2 += (item.y ** 2);
                this.regression.sums.count++;
            },

            /**
             *  Get "a" (the Y intercept value) for the regression equation.
             *
             *  a = (Σy)(Σx^2) - (Σx)(Σxy) / n(Σx^2) - (Σx)^2
             */
            calculateAValue() {
                return (
                    (this.regression.sums.y * this.regression.sums.x2) // (Σy)(Σx^2)
                    -
                    (this.regression.sums.x * this.regression.sums.xy) // (Σx)(Σxy)
                ) / (
                    (this.regression.sums.count * this.regression.sums.x2) // n(Σx^2)
                    -
                    (this.regression.sums.x ** 2) // (Σx)^2
                );
            },

            /**
             * Get "b" (the slope of the line) for the regression equation.
             *
             * b = n(Σxy) - (Σx)(Σy) / n(Σx^2) - (Σx)^2
             */
            calculateBValue() {
                return (
                    (this.regression.sums.count * this.regression.sums.xy) // n(Σxy)
                    -
                    (this.regression.sums.x * this.regression.sums.y) // (Σx)(Σy)
                ) / (
                    (this.regression.sums.count * this.regression.sums.x2) // n(Σx^2)
                    -
                    (this.regression.sums.x ** 2) // (Σx)^2
                );
            },

            /**
             * Get the Y value for an x coordinate using a linear regression equation.
             *
             * y = a + bx
             */
            calculateYValue(x) {
                return this.regression.a + (this.regression.b * x);
            },

            cloneObject(object) {
                if (object === null) {
                    return null;
                }

                return merge({}, object);
            },
            getMaxY() {
                let y = defaultYMax(),
                    series = this.series,
                    i,
                    set,
                    j,
                    item,
                    stacked = false;

                try {
                    stacked = this.options.chart.stacked;
                } catch (e) {}

                if (this.combineSeries || stacked) {
                    series = [{
                        data: this.combineSeries(series),
                    }];
                }

                for (i = 0; i < series.length; i++) {
                    set = series[i];

                    for (j = 0; j < set.data.length; j++) {
                        item = set.data[j];

                        if (y < item.y) {
                            y = item.y;
                        }
                    }
                }

                return y;
            },
        },
        mounted() {
            if (this.showTrendLine) {
                this.setTrendLine();
            }

            this.loaded = true; // added to delay rendering for data to be processed properly.
        },
    };
</script>

<style scoped>

</style>