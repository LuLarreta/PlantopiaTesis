<template>
    <section class="h-screen">
        <div class="header-map">
            <div class="back-map">
                <button class="btn-amarillo" @click="goHome">
                    <span class="material-symbols-sharp back-icon">arrow_back_ios</span>Atrás
                </button>
            </div>
            <div class="logo-map"><img src="/imgs/logo.png" alt="Logo Plantopia" class=""></div>
        </div>
        <div class="container px-4 mx-auto div-form fondo-blanco">
            <div class="header-form">
                <BaseH1 class="text-center">TU CULTIVO</BaseH1>
            </div>
            <div v-if="lastArea">
                <p class="slogan-form">Área seleccionada <span> {{
                    lastArea.areaKilometros.toFixed(3) }}
                        km²</span></p>
                <p class="mb-4 w-full text-white hidden">Coordenadas: {{ lastArea.poligons }}</p>
            </div>
            <form action="#" @submit.prevent="addNewArea">
                <div class="max-w-lg mx-auto py-4">
                    <div class="mb-6">
                        <BaseLabel for="name">Identificá tu cultivo</BaseLabel>
                        <BaseInput type="text" id="name" v-model="name" />
                    </div>
                    <div class="mb-6">
                        <BaseLabel for="weightPerHarvest">Peso proyectado (kg/km²)</BaseLabel>
                        <BaseInput type="number" id="weightPerHarvest" v-model="weightPerHarvest" />
                    </div>
                    <div class="mb-6">
                        <BaseLabel for="valuePerTon">Valor por tonelada (USD)</BaseLabel>
                        <BaseInput type="number" id="valuePerTon" v-model="valuePerTon" />
                    </div>
                    <div class="mb-6">
                        <BaseLabel for="plantationDate">Día de plantación</BaseLabel>
                        <BaseInput type="date" id="plantationDate" v-model="plantationDate" />
                    </div>
                    <div class="mb-6">
                        <BaseLabel for="harvestDate">Día de cosecha estimada</BaseLabel>
                        <BaseInput type="date" id="harvestDate" v-model="harvestDate" />
                    </div>
                    <div class="mb-2">
                        <BaseLabel for="areaColor">Color del área</BaseLabel>
                        <select class="input-base" id="areaColor" v-model="areaColor">
                            <option value="red">Rojo </option>
                            <option value="blue">Azul</option>
                            <option value="green">Verde</option>
                            <option value="yellow">Amarillo</option>
                            <option value="orange">Naranja</option>
                            <option value="purple">Violeta</option>
                            <option value="pink">Rosa</option>
                            <option value="black">Negro</option>
                            <option value="white">Blanco</option>
                            <option value="gray">Gris</option>
                            <option value="brown">Marrón</option>
                            <option value="cyan">Cian</option>
                            <option value="magenta">Magenta</option>
                        </select>
                    </div>
                    <div v-if="errorMessage" class="error">{{ errorMessage }}</div>
                    <button @click.prevent="preCalculate" class="btn-amarillo mb-6">Precalcular</button>
                    <div>
                        <Notificacion v-if="showNotification" :type="notificationType" :message="notificationMessage" />
                    </div>
                    <BaseButton :cargando="isLoading" class="mt-10">Guardar</BaseButton>
                </div>
            </form>
        </div>
    </section>
</template>

<script>
import BaseH1 from '../components/BaseH1.vue';
import BaseInput from '../components/BaseInput.vue';
import BaseLabel from '../components/BaseLabel.vue';
import BaseButton from '../components/BaseButton.vue';
import Notificacion from '../components/Notificacion.vue';

import { subscribeToAuth } from "./../service/auth.js";
import { lastAreaById, addNewDataArea, deleteArea } from "./../service/area.js";

import router from '../router/router.js';

export default {
    name: 'areas',
    components: { BaseLabel, BaseInput, BaseH1, Notificacion, BaseButton },
    data() {
        return {
            user: {
                id: null,
                email: null,
            },
            lastArea: null,
            name: null,
            weightPerHarvest: null,
            valuePerTon: null,
            areaColor: 'yellow',
            plantationDate: null,
            harvestDate: null,
            showNotification: false,
            notificationType: '',
            notificationMessage: '',
            isLoading: false,
            errorMessage: '',
        }
    },
    mounted() {
        subscribeToAuth(user => this.user = { ...user });
        this.loadLastArea();
    },
    methods: {
        async addNewArea() {
            this.isLoading = true;
            this.errorMessage = '';

            try {
                const data = {
                    name: this.name,
                    weightPerHarvest: parseFloat(this.weightPerHarvest),
                    valuePerTon: parseFloat(this.valuePerTon),
                    areaColor: this.areaColor,
                    plantationDate: this.plantationDate,
                    harvestDate: this.harvestDate
                };
                const validData = this.newAreaValidations(data);
                await addNewDataArea(this.user.id, this.lastArea.id, data);
                router.push('/user/areas');
            } catch (error) {
                switch (error.code) {
                    case 'auth/missing-name':
                        this.errorMessage = 'Por favor, ingresá el nombre del cultivo.';
                        break;
                    case 'auth/missing-weightHarvest':
                        this.errorMessage = 'Por favor, ingresá el peso proyectado.';
                        break;
                    case 'auth/invalid-weightHarvest':
                        this.errorMessage = 'Por favor, el peso proyectado tiene que ser un número válido.';
                        break;
                    case 'auth/missing-valueTon':
                        this.errorMessage = 'Por favor, ingresá el valor por tonelada.';
                        break;
                    case 'auth/invalid-valueTon':
                        this.errorMessage = 'Por favor, el valor por tonelada tiene que ser un número válido.';
                        break;
                    case 'auth/missing-area':
                        this.errorMessage = 'Falta la información del área. Por favor, volvé a marcar el área en el mapa.';
                        break;
                    case 'auth/missing-color':
                        this.errorMessage = 'Por favor, elegí un color para el área.';
                        break;
                    case 'auth/missing-plantationDate':
                        this.errorMessage = 'Por favor, ingresá el día de plantación.';
                        break;
                    case 'auth/invalid-plantationDate':
                        this.errorMessage = 'Por favor, el día de plantación tiene que ser una fecha válida.';
                        break;
                    case 'auth/missing-harvestDate':
                        this.errorMessage = 'Por favor, ingresá el día de cosecha estimada.';
                        break;
                    case 'auth/invalid-harvestDate':
                        this.errorMessage = 'Por favor, el día de cosecha estimada tiene que ser una fecha válida.';
                        break;
                }
            } finally {
                this.isLoading = false;
            }
        },
        newAreaValidations(data) {
            if (!data.name) {
                const error = new Error();
                error.code = 'auth/missing-name';
                throw error;
            } else if (!data.weightPerHarvest) {
                const error = new Error();
                error.code = 'auth/missing-weightHarvest';
                throw error;
            } else if (isNaN(data.weightPerHarvest)) {
                const error = new Error();
                error.code = 'auth/invalid-weightHarvest';
                throw error;
            } else if (!data.valuePerTon) {
                const error = new Error();
                error.code = 'auth/missing-valueTon';
                throw error;
            } else if (isNaN(data.valuePerTon)) {
                const error = new Error();
                error.code = 'auth/invalid-valueTon';
                throw error;
            } else if (!this.lastArea.id) {
                const error = new Error();
                error.code = 'auth/missing-area';
                throw error;
            } else if (!data.areaColor) {
                const error = new Error();
                error.code = 'auth/missing-color';
                throw error;
            } else if (!data.plantationDate) {
                const error = new Error();
                error.code = 'auth/missing-plantationDate';
                throw error;
            } else if (!Date.parse(data.plantationDate)) {
                const error = new Error();
                error.code = 'auth/invalid-plantationDate';
                throw error;
            } else if (!data.harvestDate) {
                const error = new Error();
                error.code = 'auth/missing-harvestDate';
                throw error;
            } else if (!Date.parse(data.harvestDate)) {
                const error = new Error();
                error.code = 'auth/invalid-harvestDate';
                throw error;
            }
        },
        async loadLastArea() {
            if (this.user.id) {
                const areaData = await lastAreaById(this.user.id);
                if (areaData) {
                    this.lastArea = areaData;
                }
            }
        },
        preCalculate() {
            const area = parseFloat(this.lastArea.areaKilometros);
            const weight = parseFloat(this.weightPerHarvest);
            const value = parseFloat(this.valuePerTon);

            if (isNaN(area) || isNaN(weight) || isNaN(value) || !area || !weight || !value) {
                this.showNotificationMessage('error', 'Ingresá valores válidos y no vacíos para área, peso y valor.');
            } else {
                this.result = ((area * weight) / 1000) * value;
                this.showNotificationMessage('success', `Ganarías un aproximado de ${this.result.toFixed(2)} USD$`);
            }
        },
        showNotificationMessage(type, message) {
            this.showNotification = true;
            this.notificationType = type;
            this.notificationMessage = message;
            setTimeout(() => {
                this.hideNotification();
            }, 5000);
        },
        hideNotification() {
            this.showNotification = false;
            this.notificationType = '';
            this.notificationMessage = '';
        },
        async goHome() {
            if (this.user.id && this.lastArea && this.lastArea.id) {
                await deleteArea(this.user.id, this.lastArea.id);
            }
            router.push('/home');
        },
    },
}
</script>
