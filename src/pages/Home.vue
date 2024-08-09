<template>
    <div class="h-screen">
        <BaseH1 class="text-center text-3xl hidden">Menú Principal</BaseH1>
        <!-- NAVBAR -->
        <nav class="container px-6 py-2 mx-auto md:flex md:justify-between md:items-center relative">
            <div class="flex items-center justify-between">
                <div @click="showMenu = !showMenu" class="flex">
                    <button type="button" class="btn_user">
                        <span class="material-symbols-sharp user_icon">
                            account_circle
                        </span>
                        <p class="ps-2">Hola {{ getEmailPrefix(user.email) }}</p>
                    </button>
                </div>
                <div class="box_user" :class="showMenu ? 'flex' : 'hidden'">
                    <ul class="py-2">
                        <li class="li_user">
                            <router-link to="/perfil" class="text-xs p-1">
                                Mi perfil
                            </router-link>
                        </li>
                        <li class="li_user">
                            <form action="#" @submit.prevent="logOuting">
                                <button class="text-xs text-red-600 p-1 " style="display: flex; align-items: center;"
                                    type="submit">Cerrar
                                    sesión<span class="material-symbols-sharp">
                                        logout
                                    </span></button>
                            </form>
                        </li>
                    </ul>
                </div>
            </div>
        </nav>

        <nav class="nav-principal">
            <ul class="lista-nav-principal">
                <li class="li-nav-principal" @click="navigateTo('/map')"><span
                        class="material-symbols-sharp">add_location_alt</span><br>Crear área</li>
                <li class="li-nav-principal" @click="navigateTo('/user/areas')"><span
                        class="material-symbols-sharp">eco</span><br>Mis cultivos</li>
                <li class="li-nav-principal"><router-link to="/home"><img src="/imgs/logo.png" alt=""
                            class="icono-home-nav"></router-link></li>
                <li class="li-nav-principal" @click="navigateTo('/user/areas/mostrarTodo')"> <span
                        class="material-symbols-sharp">travel_explore</span><br>Mis áreas</li>
                <li class="li-nav-principal" @click="navigateTo('/wiki_home')"><span
                        class="material-symbols-sharp">book_3</span><br>PlantoWiki</li>
            </ul>
        </nav>

        <!-- GRAFICO -->
        <div class="pt-5  home">
            <template v-if="!areas || areas.length === 0">
                <div class="text-center font-bold text-gray-600 mt-4">
                    <p>No tenés áreas cargadas.</p>
                </div>
            </template>
            <template v-else>
                <div class="text-center chart-container">
                    <BaseH2 class="font-bold text-2xl">Mis Estadísticas</BaseH2>
                    <div style="display: flex;">
                        <div class="estadisticas">
                            <p>Cobertura total</p>
                            <p>{{ totalCoverage }} km²</p>
                        </div>
                        <div class="estadisticas">
                            <p>Valor cosechas</p>
                            <p>USD {{ totalValueOfAreas.toFixed(2) }}</p>
                        </div>
                    </div>
                    <LineChart :labels="lineChart.labels" :data="lineChart.data" />
                    <PieChart :labels="pieChart.labels" :data="pieChart.data" :backgroundColors="pieChart.colors" />
                    <p class="exp-piechart">* Expresado en km² sobre superficie total cultivada.</p>
                </div>
            </template>
        </div>
    </div>
</template>

<script>
import BaseH1 from '../components/BaseH1.vue';
import BaseH2 from '../components/BaseH2.vue';
import LineChart from '../components/LineChart.vue';
import PieChart from '../components/PieChart.vue';

import { subscribeToAuth, logOut } from '../service/auth.js';
import { findUserAreas } from "./../service/area.js";

export default {
    name: 'home',
    components: { BaseH1, BaseH2, LineChart, PieChart },
    data() {
        return {
            showMenu: false,
            showUser: false,
            user: {
                id: null,
                email: null,
                rol: null,
            },
            areas: null,
            unsubscribeAreas: () => { },
            totalCoverage: null,
            totalValueOfAreas: 0,
            lineChart: {
                labels: ['Ene', 'Feb', 'Mar', 'Abr', 'May', 'Jun', 'Jul', 'Ago', 'Sep', 'Oct', 'Nov', 'Dic'],
                data: {
                    name: 'Mes de cosecha / USD',
                    values: new Array(12).fill(null)
                },
            },
            pieChart: {
                labels: null,
                data: null,
                colors: null,
            },
            pieChartColors: [
                'rgb(255, 99, 132)',   // red
                'rgb(255, 159, 64)',   // orange
                'rgb(255, 205, 86)',   // yellow
                'rgb(75, 192, 192)',   // green
                'rgb(54, 162, 235)',   // blue
                'rgb(153, 102, 255)',  // purple
                'rgb(201, 203, 207)'   // grey
            ],
        }
    },
    methods: {
        getEmailPrefix(email) {
            return email && email.includes('@') ? email.split('@')[0] : email;
        },
        logOuting() {
            logOut();
            this.$router.push('/login');
        },
        navigateTo(route) {
            this.$router.push(route);
        },
        getAreasByMonth() {
            const areasByMonth = Array.from({ length: 12 }, () => []);
            if (this.areas && this.areas.length > 0) {
                this.areas.forEach(area => {
                    let date;
                    date = new Date(area.harvestDate);
                    if (date) {
                        const month = date.getMonth();
                        if (areasByMonth[month]) {
                            areasByMonth[month].push(area);
                        }
                    }
                });
            }
            return areasByMonth;
        },
        getHarvestValuesUsd() {
            const areasByMonth = this.getAreasByMonth();
            const valuesByMonth = areasByMonth.map(monthAreas => {
                // Sumar el valor calculado para cada área en el mes
                const totalValues = monthAreas.reduce((total, area) => {
                    const areaValue = (((area.areaKilometers * area.weightPerHarvest) / 1000) * area.valuePerTon);
                    return total + areaValue;
                }, 0);
                // Redondear la suma a dos decimales
                return totalValues.toFixed(0);
            });
            return valuesByMonth;
        },
        getAreasByName() {
            let groupedAreas = {};
            let colorIndex = 0;

            // Recorrer todas las áreas
            this.areas.forEach(area => {
                // Verificar si ya existe una entrada para el nombre de cosecha actual
                if (groupedAreas.hasOwnProperty(area.name)) {
                    // Sumar el área actual al área acumulada
                    groupedAreas[area.name].areaKilometers += area.areaKilometers;
                } else {
                    // Si no existe, crear una nueva entrada para el nombre de cosecha
                    groupedAreas[area.name] = {
                        name: area.name,
                        areaKilometers: area.areaKilometers,
                        color: this.getColor(colorIndex)  // Función para obtener un color aleatorio
                    };
                    colorIndex++;
                }
            });

            // Extraer los resultados en arreglos separados
            let names = Object.keys(groupedAreas);
            let areas = names.map(name => groupedAreas[name].areaKilometers);
            let colors = names.map(name => groupedAreas[name].color);

            this.pieChart.labels = names;
            this.pieChart.data = areas;
            this.pieChart.colors = colors;
        },
        getColor(i) {
            const color = this.pieChartColors[i % this.pieChartColors.length];
            return color;
        },
        calculateTotalCoverage() {
            if (this.areas && this.areas.length > 0) {
                return this.areas.reduce((total, area) => total + area.areaKilometers, 0).toFixed(3);
            } else {
                return 0;
            }
        },
        calculateTotalValueOfAreas() {
            if (this.areas && this.areas.length > 0) {
                return this.areas.reduce((total, area) => total + (((area.areaKilometers * area.weightPerHarvest) / 1000) * area.valuePerTon), 0);
            } else {
                return 0;
            }
        }
    },
    mounted() {
        subscribeToAuth(user => {
            this.user = { ...user }
        });
        this.unsubscribeAreas = findUserAreas(this.user.id, (areas) => {
            this.areas = areas;
            this.totalCoverage = this.calculateTotalCoverage();
            this.totalValueOfAreas = this.calculateTotalValueOfAreas();
            this.lineChart.data.values = this.getHarvestValuesUsd();
            this.pieChart.values = this.getAreasByName();
        });
    },
}
</script>

<style scoped>
h2 {
    text-transform: uppercase;
    color: #000;
}
</style>