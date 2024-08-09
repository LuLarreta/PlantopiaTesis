<template>
    <div class="piechart">
        <canvas ref="pieChart"></canvas>
    </div>
</template>

<script>
import { ref, onMounted } from 'vue';
import Chart from 'chart.js/auto';

export default {
    name: 'PieChart',
    props: {
        labels: {
            type: Array,
            required: true
        },
        data: {
            type: Array,
            required: true
        },
        backgroundColors: {
            type: Array,
            required: true
        }
    },
    setup(props) {
        const pieChart = ref(null);
        let chartInstance = null;

        onMounted(() => {
            const ctx = pieChart.value.getContext('2d');
            chartInstance = new Chart(ctx, {
                type: 'pie',
                data: {
                    labels: props.labels,
                    datasets: [{
                        label: 'Dataset',
                        data: props.data,
                        backgroundColor: props.backgroundColors,
                        hoverOffset: 6
                    }]
                },
                options: {
                    responsive: true,
                    circumferencePercentage: 50,
                    plugins: {
                        legend: {
                            position: 'top',
                        },
                        tooltip: {
                            callbacks: {

                            }
                        }
                    }
                }
            });
        });

        return { pieChart };
    }
}
</script>

<style scoped>
div {
    display: flex;
    justify-content: center;
    max-width: 600px;
    padding-bottom: 10px;
}
</style>