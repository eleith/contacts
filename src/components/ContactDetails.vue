<!--
  - @copyright Copyright (c) 2018 John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @author John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @license GNU AGPL version 3 or any later version
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<AppContentDetails>
		<!-- nothing selected or contact not found -->
		<EmptyContent v-if="!contact" icon="icon-contacts">
			{{ t('contacts', 'No contact selected') }}
			<template #desc>
				{{ t('contacts', 'Select a contact on the list to begin') }}
			</template>
		</EmptyContent>

		<template v-else>
			<!-- contact header -->
			<header class="contact-header">
				<!-- avatar and upload photo -->
				<ContactAvatar
					:contact="contact"
					@update-local-contact="updateLocalContact" />
				<!-- QUESTION: is it better to pass contact as a prop or get it from the store inside
				contact-avatar ?  :avatar="contact.photo"-->

				<!-- fullname, org, title -->
				<div class="contact-header__infos">
					<h2>
						<input id="contact-fullname"
							ref="fullname"
							v-model="contact.fullName"
							:readonly="contact.addressbook.readOnly"
							:placeholder="t('contacts', 'Name')"
							type="text"
							autocomplete="off"
							autocorrect="off"
							spellcheck="false"
							name="fullname"
							@input="debounceUpdateContact"
							@click="selectInput">
					</h2>
					<div id="details-org-container">
						<input id="contact-org"
							v-model="contact.org"
							:readonly="contact.addressbook.readOnly"
							:placeholder="t('contacts', 'Company')"
							type="text"
							autocomplete="off"
							autocorrect="off"
							spellcheck="false"
							name="org"
							@input="debounceUpdateContact">
						<input id="contact-title"
							v-model="contact.title"
							:readonly="contact.addressbook.readOnly"
							:placeholder="t('contacts', 'Title')"
							type="text"
							autocomplete="off"
							autocorrect="off"
							spellcheck="false"
							name="title"
							@input="debounceUpdateContact">
					</div>
				</div>

				<!-- actions -->
				<div class="contact-header__actions">
					<!-- warning message -->
					<a v-if="loadingUpdate || warning"
						v-tooltip.bottom="{
							content: warning ? warning.msg : '',
							trigger: 'hover focus'
						}"
						:class="{'icon-loading-small': loadingUpdate,
							[`${warning.icon}`]: warning}"
						class="header-icon"
						@click="onWarningClick" />

					<!-- conflict message -->
					<div v-if="conflict"
						v-tooltip="{
							content: conflict,
							show: true,
							trigger: 'manual',
						}"
						class="header-icon header-icon--pulse icon-history"
						@click="refreshContact" />

					<!-- repaired contact message -->
					<div v-if="fixed"
						v-tooltip="{
							content: t('contacts', 'This contact was broken and received a fix. Please review the content and click here to save it.'),
							show: true,
							trigger: 'manual',
						}"
						class="header-icon header-icon--pulse icon-up"
						@click="updateContact" />

					<!-- menu actions -->
					<Actions ref="actions"
						class="header-menu"
						menu-align="right"
						:open.sync="openedMenu">
						<ActionLink :href="contact.url"
							:download="`${contact.displayName}.vcf`"
							icon="icon-download">
							{{ t('contacts', 'Download') }}
						</ActionLink>
						<!-- user can clone if there is at least one option available -->
						<ActionButton v-if="isReadOnly && addressbooksOptions.length > 0"
							ref="cloneAction"
							:close-after-click="true"
							icon="icon-clone"
							@click="cloneContact">
							{{ t('contacts', 'Clone contact') }}
						</ActionButton>
						<ActionButton icon="icon-qrcode" @click="showQRcode">
							{{ t('contacts', 'Generate QR Code') }}
						</ActionButton>
						<ActionButton v-if="!isReadOnly" icon="icon-delete" @click="deleteContact">
							{{ t('contacts', 'Delete') }}
						</ActionButton>
					</Actions>
				</div>

				<!-- qrcode -->
				<Modal v-if="qrcode"
					id="qrcode-modal"
					:clear-view-delay="-1"
					:title="contact.displayName"
					@close="closeQrModal">
					<img :src="`data:image/svg+xml;base64,${qrcode}`"
						:alt="t('contacts', 'Contact vCard as qrcode')"
						class="qrcode"
						width="400">
				</Modal>

				<!-- pick addressbook when cloning contact -->
				<Modal v-if="showPickAddressbookModal"
					id="pick-addressbook-modal"
					:clear-view-delay="-1"
					:title="t('contacts', 'Pick an address book')"
					@close="closePickAddressbookModal">
					<Multiselect ref="pickAddressbook"
						v-model="pickedAddressbook"
						:allow-empty="false"
						:options="addressbooksOptions"
						:placeholder="t('contacts', 'Select address book')"
						track-by="id"
						label="name" />
					<button @click="closePickAddressbookModal">
						{{ t('contacts', 'Cancel') }}
					</button>
					<button class="primary" @click="cloneContact">
						{{ t('contacts', 'Clone contact') }}
					</button>
				</Modal>
			</header>

			<!-- contact details loading -->
			<section v-if="loadingData" class="icon-loading contact-details" />

			<!-- contact details -->
			<section v-else
				v-masonry="contactDetailsSelector"
				class="contact-details"
				:fit-width="true"
				item-selector=".property-masonry"
				:transition-duration="0">
				<!-- properties iteration -->
				<!-- using contact.key in the key and index as key to avoid conflicts between similar data and exact key -->
				<!-- passing the debounceUpdateContact so that the contact-property component contains the function
					and allow us to use it on the rfcProps since the scope is forwarded to the actions -->
				<div v-for="(properties, name) in groupedProperties"
					:key="name"
					v-masonry-tile
					class="property-masonry">
					<ContactProperty v-for="(property, index) in properties"
						:key="`${index}-${contact.key}-${property.name}`"
						:is-first-property="index===0"
						:is-last-property="index === properties.length - 1"
						:property="property"
						:contact="contact"
						:local-contact="localContact"
						:update-contact="debounceUpdateContact"
						@resize="redrawMasonry" />
				</div>

				<!-- addressbook change select - no last property because class is not applied here,
					empty property because this is a required prop on regular property-select. But since
					we are hijacking this... (this is supposed to be used with a ICAL.property, but to avoid code
					duplication, we created a fake propModel and property with our own options here) -->
				<PropertySelect v-masonry-tile
					:prop-model="addressbookModel"
					:value.sync="addressbook"
					:is-first-property="true"
					:is-last-property="true"
					:property="{}"
					class="property-masonry property--addressbooks property--last property--without-actions" />

				<!-- Groups always visible -->
				<PropertyGroups v-masonry-tile
					:prop-model="groupsModel"
					:value.sync="groups"
					:contact="contact"
					:is-read-only="isReadOnly"
					class="property-masonry property--groups property--last" />

				<!-- new property select -->
				<AddNewProp v-if="!isReadOnly"
					v-masonry-tile
					:contact="contact"
					class="property-masonry" />

				<!-- Last modified-->
				<PropertyRev v-if="contact.rev" :value="contact.rev" />
			</section>
		</template>
	</AppContentDetails>
</template>

<script>
import { showError } from '@nextcloud/dialogs'
import { stringify } from 'ical.js'
import debounce from 'debounce'
import PQueue from 'p-queue'
import qr from 'qr-image'
import Vue from 'vue'
import { VueMasonryPlugin } from 'vue-masonry'

import ActionButton from '@nextcloud/vue/dist/Components/ActionButton'
import ActionLink from '@nextcloud/vue/dist/Components/ActionLink'
import Actions from '@nextcloud/vue/dist/Components/Actions'
import AppContentDetails from '@nextcloud/vue/dist/Components/AppContentDetails'
import Modal from '@nextcloud/vue/dist/Components/Modal'
import Multiselect from '@nextcloud/vue/dist/Components/Multiselect'

import rfcProps from '../models/rfcProps'
import validate from '../services/validate'

import AddNewProp from './ContactDetails/ContactDetailsAddNewProp'
import ContactAvatar from './ContactDetails/ContactDetailsAvatar'
import ContactProperty from './ContactDetails/ContactDetailsProperty'
import EmptyContent from './EmptyContent'
import PropertyGroups from './Properties/PropertyGroups'
import PropertyRev from './Properties/PropertyRev'
import PropertySelect from './Properties/PropertySelect'

Vue.use(VueMasonryPlugin)
const updateQueue = new PQueue({ concurrency: 1 })

export default {
	name: 'ContactDetails',

	components: {
		ActionButton,
		ActionLink,
		Actions,
		AddNewProp,
		AppContentDetails,
		ContactAvatar,
		ContactProperty,
		EmptyContent,
		Modal,
		Multiselect,
		PropertyGroups,
		PropertyRev,
		PropertySelect,
	},

	props: {
		contactKey: {
			type: String,
			default: undefined,
		},
	},

	data() {
		return {
			// if true, the local contact have been fixed and requires a push
			fixed: false,
			/**
			 * Local off-store clone of the selected contact for edition
			 * because we can't edit contacts data outside the store.
			 * Every change will be dispatched and updated on the real
			 * store contact after a debounce.
			 */
			localContact: undefined,
			loadingData: true,
			loadingUpdate: false,
			openedMenu: false,
			qrcode: '',
			showPickAddressbookModal: false,
			pickedAddressbook: null,

			contactDetailsSelector: '.contact-details',
		}
	},

	computed: {
		isReadOnly() {
			if (this.contact.addressbook) {
				return this.contact.addressbook.readOnly
			}
			return false
		},

		/**
		 * Warning messages
		 *
		 * @returns {Object|boolean}
		 */
		warning() {
			if (!this.contact.dav) {
				return {
					icon: 'icon-error header-icon--pulse',
					msg: t('contacts', 'This contact is not yet synced. Edit it to save it to the server.'),
				}
			} else if (this.isReadOnly) {
				return {
					icon: 'icon-eye',
					msg: t('contacts', 'This contact is in read-only mode. You do not have permission to edit this contact.'),
				}
			}
			return false
		},

		/**
		 * Conflict message
		 *
		 * @returns {string|boolean}
		 */
		conflict() {
			if (this.contact.conflict) {
				return t('contacts', 'The contact you were trying to edit has changed. Please manually refresh the contact. Any further edits will be discarded.')
			}
			return false
		},

		/**
		 * Contact properties copied and sorted by rfcProps.fieldOrder
		 *
		 * @returns {Array}
		 */
		sortedProperties() {
			return this.localContact.properties
				.slice(0)
				.sort((a, b) => {
					const nameA = a.name.split('.').pop()
					const nameB = b.name.split('.').pop()
					return rfcProps.fieldOrder.indexOf(nameA) - rfcProps.fieldOrder.indexOf(nameB)
				})
		},

		/**
		 * Contact properties filtered and grouped by rfcProps.fieldOrder
		 *
		 * @returns {Object}
		 */
		groupedProperties() {
			return this.sortedProperties
				.reduce((list, property) => {
					// If there is no component to display this prop, ignore it
					if (!this.canDisplay(property)) {
						return list
					}

					// Init if needed
					if (!list[property.name]) {
						list[property.name] = []
					}

					list[property.name].push(property)
					return list
				}, {})
		},

		/**
		 * Fake model to use the propertySelect component
		 *
		 * @returns {Object}
		 */
		addressbookModel() {
			return {
				readableName: t('contacts', 'Address book'),
				icon: 'icon-address-book',
				options: this.addressbooksOptions,
			}
		},

		/**
		 * Usable addressbook object linked to the local contact
		 *
		 * @param {string} [addressbookId] set the addressbook id
		 * @returns {string}
		 */
		addressbook: {
			get() {
				return this.contact.addressbook.id
			},
			set(addressbookId) {
				this.moveContactToAddressbook(addressbookId)
			},
		},

		/**
		 * Fake model to use the propertyGroups component
		 *
		 * @returns {Object}
		 */
		groupsModel() {
			return {
				readableName: t('contacts', 'Groups'),
				icon: 'icon-contacts',
			}
		},

		/**
		 * Usable groups object linked to the local contact
		 *
		 * @param {string[]} data An array of groups
		 * @returns {Array}
		 */
		groups: {
			get() {
				return this.contact.groups
			},
			set(data) {
				this.contact.groups = data
				this.debounceUpdateContact()
			},
		},

		/**
		 * Store getters filtered and mapped to usable object
		 * This is the list of addressbooks that are available to write
		 *
		 * @returns {Array}
		 */
		addressbooksOptions() {
			return this.addressbooks
				.filter(addressbook => !addressbook.readOnly && addressbook.enabled)
				.map(addressbook => {
					return {
						id: addressbook.id,
						name: addressbook.displayName,
					}
				})
		},

		// store getter
		addressbooks() {
			return this.$store.getters.getAddressbooks
		},
		contact() {
			return this.$store.getters.getContact(this.contactKey)
		},
	},

	watch: {
		contact(newContact, oldContact) {
			if (this.contactKey && newContact !== oldContact) {
				this.selectContact(this.contactKey)
			}

			// Reflow grid
			this.$redrawVueMasonry(this.contactDetailsSelector)
		},
	},

	updated() {
		this.$redrawVueMasonry(this.contactDetailsSelector)
	},

	beforeMount() {
		// load the desired data if we already selected a contact
		if (this.contactKey) {
			this.selectContact(this.contactKey)
		}

		// capture ctrl+s
		document.addEventListener('keydown', this.onCtrlSave)
	},

	beforeDestroy() {
		// unbind capture ctrl+s
		document.removeEventListener('keydown', this.onCtrlSave)
	},

	methods: {
		/**
		 * Send the local clone of contact to the store
		 */
		async updateContact() {
			this.fixed = false
			this.loadingUpdate = true
			await this.$store.dispatch('updateContact', this.localContact)
			this.loadingUpdate = false

			// if we just created the contact, we need to force update the
			// localContact to match the proper store contact
			if (!this.localContact.dav) {
				console.debug('New contact synced!', this.localContact)
				// fetching newly created & storred contact
				const contact = this.$store.getters.getContact(this.localContact.key)
				await this.updateLocalContact(contact)
			}
		},

		/**
		 * Debounce the contact update for the header props
		 * photo, fn, org, title
		 */
		debounceUpdateContact: debounce(function(e) {
			updateQueue.add(this.updateContact)
		}, 500),

		// menu handling
		closeMenu() {
			this.openedMenu = false
		},
		toggleMenu() {
			this.openedMenu = !this.openedMenu
		},

		/**
		 * Generate a qrcode for the contact
		 */
		showQRcode() {
			const jCal = this.contact.jCal.slice(0)
			// do not encode photo
			jCal[1] = jCal[1].filter(props => props[0] !== 'photo')

			const data = stringify(jCal)
			if (data.length > 0) {
				this.qrcode = btoa(qr.imageSync(data, { type: 'svg' }))
			}
		},

		/**
		 * Select the text in the input if it is still set to 'new Contact'
		 */
		selectInput() {
			if (this.$refs.fullname && this.$refs.fullname.value === t('contacts', 'New contact')) {
				this.$refs.fullname.select()
			}
		},

		/**
		 * Select a contact, and update the localContact
		 * Fetch updated data if necessary
		 * Scroll to the selected contact if exists
		 *
		 * @param {string} key the contact key
		 */
		async selectContact(key) {
			this.loadingData = true

			// local version of the contact
			const contact = this.$store.getters.getContact(key)

			if (contact) {
				// if contact exists AND if exists on server
				if (contact.dav) {
					try {
						await this.$store.dispatch('fetchFullContact', { contact })
						// clone to a local editable variable
						await this.updateLocalContact(contact)
					} catch (error) {
						if (error.name === 'ParserError') {
							showError(t('contacts', 'Syntax error. Cannot open the contact.'))
						} else if (error.status === 404) {
							showError(t('contacts', 'The contact doesn\'t exists anymore on the server.'))
						} else {
							showError(t('contacts', 'Unable to retrieve the contact from the server, please check your network connection.'))
						}
						console.error(error)
						// trigger a local deletion from the store only
						this.$store.dispatch('deleteContact', { contact: this.contact, dav: false })
					}
				} else {
					// clone to a local editable variable
					await this.updateLocalContact(contact)
				}
			}

			this.loadingData = false
		},

		/**
		 * Dispatch contact deletion request
		 */
		deleteContact() {
			this.$store.dispatch('deleteContact', { contact: this.contact })
		},

		/**
		 * Move contact to the specified addressbook
		 *
		 * @param {string} addressbookId the desired addressbook ID
		 */
		async moveContactToAddressbook(addressbookId) {
			const addressbook = this.addressbooks.find(search => search.id === addressbookId)
			this.loadingUpdate = true
			if (addressbook) {
				try {
					const contact = await this.$store.dispatch('moveContactToAddressbook', {
						// we need to use the store contact, not the local contact
						// using this.contact and not this.localContact
						contact: this.contact,
						addressbook,
					})
					// select the contact again
					this.$router.push({
						name: 'contact',
						params: {
							selectedGroup: this.$route.params.selectedGroup,
							selectedContact: contact.key,
						},
					})
				} catch (error) {
					console.error(error)
					showError(t('contacts', 'An error occurred while trying to move the contact'))
				} finally {
					this.loadingUpdate = false
				}
			}
		},

		/**
		 * Copy contact to the specified addressbook
		 *
		 * @param {string} addressbookId the desired addressbook ID
		 */
		async copyContactToAddressbook(addressbookId) {
			const addressbook = this.addressbooks.find(search => search.id === addressbookId)
			this.loadingUpdate = true
			if (addressbook) {
				try {
					const contact = await this.$store.dispatch('copyContactToAddressbook', {
						// we need to use the store contact, not the local contact
						// using this.contact and not this.localContact
						contact: this.contact,
						addressbook,
					})
					// select the contact again
					this.$router.push({
						name: 'contact',
						params: {
							selectedGroup: this.$route.params.selectedGroup,
							selectedContact: contact.key,
						},
					})
				} catch (error) {
					console.error(error)
					showError(t('contacts', 'An error occurred while trying to copy the contact'))
				} finally {
					this.loadingUpdate = false
				}
			}
		},

		/**
		 * Refresh the data of a contact
		 */
		refreshContact() {
			this.$store.dispatch('fetchFullContact', { contact: this.contact, etag: this.conflict })
				.then(() => {
					this.contact.conflict = false
				})
		},

		// reset the current qrcode
		closeQrModal() {
			this.qrcode = ''
		},

		/**
		 *  Update this.localContact and set this.fixed
		 *
		 * @param {Contact} contact the contact to clone
		 */
		async updateLocalContact(contact) {
			// create empty contact and copy inner data
			const localContact = Object.assign(
				Object.create(Object.getPrototypeOf(contact)),
				contact
			)

			this.fixed = validate(localContact)

			this.localContact = localContact
		},

		onCtrlSave(e) {
			if (e.keyCode === 83 && (navigator.platform.match('Mac') ? e.metaKey : e.ctrlKey)) {
				e.preventDefault()
				this.debounceUpdateContact()
			}
		},

		/**
		 * Clone the current contact to another addressbook
		 */
		async cloneContact() {
			// only one addressbook, let's clone it there
			if (this.pickedAddressbook && this.addressbooks.find(addressbook => addressbook.id === this.pickedAddressbook.id)) {
				console.debug('Cloning contact to', this.pickedAddressbook.name)
				await this.copyContactToAddressbook(this.pickedAddressbook.id)
				this.closePickAddressbookModal()
			} else if (this.addressbooksOptions.length === 1) {
				console.debug('Cloning contact to', this.addressbooksOptions[0].name)
				await this.copyContactToAddressbook(this.addressbooksOptions[0].id)
			} else {
				this.showPickAddressbookModal = true
			}
		},

		closePickAddressbookModal() {
			this.showPickAddressbookModal = false
			this.pickedAddressbook = null
		},

		/**
		 * The user clicked the warning icon
		 */
		onWarningClick() {
			// if the user clicked the readonly icon, let's focus the clone button
			if (this.isReadOnly && this.addressbooksOptions.length > 0) {
				this.openedMenu = true
				this.$nextTick(() => {
					// focus the clone button
					this.$refs.actions.onMouseFocusAction({ target: this.$refs.cloneAction.$el })
				})
			}
		},

		/**
		 * Should display the property
		 *
		 * @param {Property} property the property to check
		 * @returns {boolean}
		 */
		canDisplay(property) {
			const propModel = rfcProps.properties[property.name]
			const propType = propModel && propModel.force
				? propModel.force
				: property.getDefaultType()

			return propModel && propType !== 'unknown'
		},

		/**
		 * Redraw Masonry
		 */
		redrawMasonry() {
			this.$redrawVueMasonry(this.contactDetailsSelector)
		},
	},
}
</script>

<style lang="scss">
.app-content-details {
	flex: 1 1 100%;
	min-width: 0;

	// Header with avatar, name, position, actions...
	header {
		display: flex;
		align-items: center;
		padding: 50px 0 20px;
		font-weight: bold;

		// ORG-TITLE-NAME
		.contact-header__infos {
			display: flex;
			flex: 1 1 auto; // shrink avatar before this one
			flex-direction: column;
			h2,
			#details-org-container {
				display: flex;
				flex-wrap: wrap;
				margin: 0;
			}
			input {
				overflow: hidden;
				flex: 1 1;
				min-width: 100px;
				max-width: 100%;
				margin: 0;
				padding: 4px 5px;
				white-space: nowrap;
				text-overflow: ellipsis;
				border: none;
				background: transparent;
				font-size: inherit;
				&#contact-fullname {
					font-weight: bold;
				}
			}
			#contact-org:placeholder-shown {
				max-width: 20%;
			}
		}

		// ACTIONS
		.contact-header__actions {
			position: relative;
			display: flex;
			.header-menu {
				margin-right: 10px;
			}
			.header-icon {
				width: 44px;
				height: 44px;
				padding: 14px;
				cursor: pointer;
				opacity: .7;
				border-radius: 22px;
				background-size: 16px;
				&:hover,
				&:focus {
					opacity: 1;
				}
				&.header-icon--pulse {
					width: 16px;
					height: 16px;
					margin: 8px;
				}
			}
		}
	}

	// List of all properties
	section.contact-details {
		margin: 0 auto;

		.property-masonry {
			width: 350px;
		}

		.property--rev {
			position: fixed;
			right: 22px;
			bottom: 0;
			height: 44px;
			opacity: .5;
			color: var(--color-text-lighter);
			line-height: 44px;
		}
	}

	#qrcode-modal {
		.modal-container {
			display: flex;
			padding: 10px;
			background-color: #fff;
			.qrcode {
				max-width: 100%;
			}
		}
	}

	#pick-addressbook-modal {
		.modal-container {
			display: flex;
			overflow: visible;
			flex-wrap: wrap;
			justify-content: space-evenly;
			margin-bottom: 20px;
			padding: 10px;
			background-color: #fff;
			.multiselect {
				flex: 1 1 100%;
				width: 100%;
				margin-bottom: 20px;
			}
		}
	}
}

</style>
