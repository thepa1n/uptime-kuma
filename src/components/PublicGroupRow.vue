<template>
    <div class="row">
        <div class="col-6 small-padding">
            <div class="info">
                <font-awesome-icon v-if="showDragRemove" icon="arrows-alt-v" class="action drag me-3" />

                <font-awesome-icon v-if="showDragRemove" icon="times" class="action remove me-3" @click.stop="onRemoveClick" />

                <span v-if="displayIndicator" class="nested-indicator">{{ displayIndicator }}</span>

                <font-awesome-icon
                    v-if="element.type === 'group' && element.childrenList && element.childrenList.length"
                    :icon="isExpanded ? 'chevron-down' : 'chevron-right'"
                    class="group-toggle-icon me-2"
                    @click.stop="toggleGroupExpand"
                />

                <Uptime :monitor="element" type="24" :pill="true" />

                <a v-if="showLink(element)" :href="element.url" class="item-name" target="_blank" rel="noopener noreferrer" :data-testid="nameTestId">
                    {{ element.name }}
                </a>

                <p v-else class="item-name" :data-testid="nameTestId"> {{ element.name }} </p>

                <span title="Setting">
                    <font-awesome-icon
                        v-if="onSettings"
                        :class="{'link-active': true, 'btn-link': true}"
                        icon="cog" class="action me-3"
                        @click.stop="onSettingsClick"
                    />
                </span>
            </div>
            <div class="extra-info">
                <div v-if="showCertificateExpiry && element.certExpiryDaysRemaining">
                    <Tag :item="{name: $t('Cert Exp.'), value: formattedCertExpiryMessage(element), color: certExpiryColor(element)}" :size="'sm'" />
                </div>
                <div v-if="showTags && element.tags">
                    <Tag v-for="tag in element.tags" :key="tag" :item="tag" :size="'sm'" />
                </div>
            </div>
        </div>
        <div :key="heartbeatKey" class="col-6">
            <HeartbeatBar size="mid" :monitor-id="element.id" />
        </div>
    </div>

    <transition name="slide-fade-up">
        <div v-if="element.type === 'group' && element.childrenList && element.childrenList.length && isExpanded" class="nested-monitors">
            <div v-for="child in element.childrenList" :key="child.id" class="item nested-item" data-testid="nested-monitor">
                <PublicGroupRow
                    :element="child"
                    :show-tags="showTags"
                    :show-certificate-expiry="showCertificateExpiry"
                    :edit-mode="editMode"
                    :heartbeat-key="$root.userHeartbeatBar"
                    name-test-id="nested-monitor-name"
                    :depth="depth + 1"
                />
            </div>
        </div>
    </transition>
</template>

<script>
import HeartbeatBar from "./HeartbeatBar.vue";
import Uptime from "./Uptime.vue";
import Tag from "./Tag.vue";

export default {
    name: "PublicGroupRow",
    components: {
        Uptime,
        HeartbeatBar,
        Tag,
    },
    props: {
        element: { type: Object,
            required: true },
        showDragRemove: { type: Boolean,
            default: false },
        onRemove: { type: Function,
            default: null },
        onSettings: { type: Function,
            default: null },
        showTags: { type: Boolean,
            default: false },
        showCertificateExpiry: { type: Boolean,
            default: false },
        editMode: { type: Boolean,
            default: false },
        heartbeatKey: { type: [ String, Number ],
            default: null },
        indicator: { type: String,
            default: "" },
        depth: { type: Number,
            default: 0 },
        nameTestId: { type: String,
            default: "monitor-name" },
    },
    data() {
        return {
            isExpanded: false,
        };
    },
    computed: {
        displayIndicator() {
            if (this.indicator) {
                return this.indicator;
            }

            if (this.depth > 0) {
                return "└─";
            }

            return "";
        }
    },
    beforeMount() {
        try {
            const storage = window.localStorage.getItem("publicGroupRowExpanded");
            if (storage) {
                const obj = JSON.parse(storage);
                if (obj[`monitor_${this.element.id}`] != null) {
                    this.isExpanded = obj[`monitor_${this.element.id}`];
                }
            }
        } catch (e) {
            // ignore
        }
    },
    methods: {
        /**
         * Toggle expand/collapse state for a group row
         * @returns {void}
         */
        toggleGroupExpand() {
            this.isExpanded = !this.isExpanded;

            try {
                let storage = window.localStorage.getItem("publicGroupRowExpanded");
                let storageObject = {};
                if (storage !== null) {
                    storageObject = JSON.parse(storage);
                }
                storageObject[`monitor_${this.element.id}`] = this.isExpanded;
                window.localStorage.setItem("publicGroupRowExpanded", JSON.stringify(storageObject));
            } catch (e) {
                // ignore
            }
        },
        onRemoveClick(e) {
            e.stopPropagation();
            if (this.onRemove) {
                this.onRemove();
            }
        },
        onSettingsClick(e) {
            e.stopPropagation();
            if (this.onSettings) {
                this.onSettings();
            }
        },

        /**
         * Should a link to the monitor be shown?
         * Attempts to guess if a link should be shown based upon if
         * sendUrl is set and if the URL is default or not.
         * @param {object} element Monitor to check
         * @param {boolean} ignoreSendUrl Should the presence of the sendUrl
         * property be ignored. This will only work in edit mode.
         * @returns {boolean} Should the link be shown
         */
        showLink(element, ignoreSendUrl = false) {
            // We must check if there are any elements in monitorList to
            // prevent undefined errors if it hasn't been loaded yet
            if (this.editMode && ignoreSendUrl && Object.keys(this.$root.monitorList || {}).length) {
                const m = this.$root.monitorList[element.id];
                return m && (m.type === "http" || m.type === "keyword" || m.type === "json-query");
            }

            return element.sendUrl && element.url && element.url !== "https://";
        },

        /**
         * Returns formatted certificate expiry or Bad cert message
         * @param {object} element Element to show expiry for
         * @returns {string} Certificate expiry message
         */
        formattedCertExpiryMessage(element) {
            if (element?.validCert && element?.certExpiryDaysRemaining) {
                return element.certExpiryDaysRemaining + " " + this.$tc("day", element.certExpiryDaysRemaining);
            } else if (element?.validCert === false) {
                return this.$t("noOrBadCertificate");
            } else {
                return this.$t("Unknown") + " " + this.$tc("day", 2);
            }
        },

        /**
         * Returns certificate expiry color based on days remaining
         * @param {object} element Element to show expiry for
         * @returns {string} Color for certificate expiry
         */
        certExpiryColor(element) {
            if (element?.validCert && element.certExpiryDaysRemaining > 7) {
                return "#059669";
            }

            return "#DC2626";
        },
    }
};
</script>

<style lang="scss" scoped>
@import "../assets/vars";

.nested-indicator {
    color: #999;
    margin-right: 5px;
    font-family: monospace;
}

.group-toggle-icon {
    cursor: pointer;
    color: #666;
    transition: color 0.2s ease;
    font-size: 0.9em;
}

.group-toggle-icon:hover {
    color: $primary;
}

.dark .group-toggle-icon {
    color: #aaa;
}

.dark .group-toggle-icon:hover {
    color: $primary;
}

.item-name {
    padding-left: 5px;
    padding-right: 5px;
    margin: 0;
    display: inline-block;
}

.nested-item {
    background-color: rgba(0, 0, 0, 0.05);
    border-left: 3px solid rgba(0, 0, 0, 0.1);
}

.dark .nested-item {
    background-color: rgba(255, 255, 255, 0.02);
    border-left: 3px solid rgba(255, 255, 255, 0.1);
}

.nested-monitors {
    margin-left: 0;
    margin-top: 0;
    animation: slide-down 0.3s ease-out;
    overflow: hidden;
}

@keyframes slide-down {
    from {
        opacity: 0;
        max-height: 0;
    }

    to {
        opacity: 1;
        max-height: 2000px;
    }
}

</style>
