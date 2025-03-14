<template>
    <v-app id="inspire">
        <v-app-bar color="indigo">
            <v-toolbar-title>{{ globalStore.title }}</v-toolbar-title>
        </v-app-bar>

        <v-main>
            <v-container class="fill-height" fluid>
                <v-row align="center" justify="center">
                    <v-col class="text-center">
                        <span>{{ message }}</span>
                        <br />
                        <v-btn class="ma-2" variant="text" icon color="pink-lighten-1" @click="handleHeartClick">
                            <v-icon>mdi-heart</v-icon>
                        </v-btn>
                        <br />
                        <v-btn class="ma-2" color="primary" @click="connectMQTT"> Connect to MQTT </v-btn>
                        <div v-if="mqttStatus">Connection Status: {{ mqttStatus }}</div>
                    </v-col>
                </v-row>
            </v-container>
        </v-main>
        <v-footer color="indigo" app>
            <span class="text-white">&copy; {{ new Date().getFullYear() }}</span>
        </v-footer>
    </v-app>
</template>

<script>
import { mapStores } from "pinia";
import mqtt from "mqtt";
import { requirejs, widget } from "@widget-lab/3ddashboard-utils";
import { useGlobalStore } from "@/store/global";

console.debug(widget);

export default {
    name: "App",
    data() {
        return {
            message: "hello",
            mqttClient: null,
            mqttStatus: null,
            clickCount: 0,
            platformAPI: {}
        };
    },

    // imporat to mount
    async mounted() {
        // Subscribe to world click events
        this.platformAPI = await requirejs("DS/PlatformAPI/PlatformAPI");
        this.platformAPI.subscribe("3DEXPERIENCity.OnWorldClick", res => console.log(res));
    },
    beforeUnmount() {
        // Cleanup subscription when component is destroyed
        widget.PublicAPI.OnWorldClick.unsubscribe(this.handleWorldClick);
    },
    computed: {
        ...mapStores(useGlobalStore)
    },
    methods: {
        handleWorldClick(event) {
            console.log("World Click Event:", {
                position: event.position,
                normal: event.normal,
                button: event.button,
                pickedObject: event.pickedObject
            });
        },
        handleHeartClick() {
            this.clickCount++;
            this.message = `Clicked ${this.clickCount} times!`;

            if (this.mqttClient && this.mqttClient.connected) {
                this.mqttClient.publish(
                    "george/sensor",
                    JSON.stringify({
                        action: "heart_click",
                        count: this.clickCount,
                        timestamp: new Date().toISOString()
                    })
                );
            } else {
                console.warn("MQTT not connected. Cannot publish message.");
                this.mqttStatus = "Not connected to MQTT";
            }
        },
        connectMQTT() {
            if (this.mqttClient) {
                this.mqttClient.end();
            }

            const options = {
                protocol: "wss",
                hostname: "test.mosquitto.org",
                port: 8081,
                clientId: "vue-client-" + Math.random().toString(16).substr(2, 8)
            };

            this.mqttStatus = "Connecting...";
            this.mqttClient = mqtt.connect(options);

            this.mqttClient.on("connect", () => {
                console.log("Connected to MQTT broker");
                this.mqttStatus = "Connected";
                this.mqttClient.subscribe("george/sensor", err => {
                    if (!err) {
                        console.log("Subscribed to george/sensor");
                    }
                });
            });

            this.mqttClient.on("message", (topic, message) => {
                console.log(`Received message from ${topic}: ${message.toString()}`);
            });

            this.mqttClient.on("error", error => {
                console.error("MQTT Error:", error);
                this.mqttStatus = "Error: " + error.message;
            });
        }
    }
};
</script>
