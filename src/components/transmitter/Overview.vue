<template>
	<div class="container">
		<div class="row">
			<div class="col-lg-12">
				<div class="page-header">
					<h1>Transmitters</h1>
				</div>
			</div>
		</div>

		<div class="row">
			<div class="col-lg-9">
				<h2>All Transmitters
					<i class="fa fa-refresh fa-fw" :class="{ 'fa-spin': running }" @click="loadData"></i>
				</h2>

				<info-error :message="errorMessage"></info-error>

				<tablegrid v-if="table.rows" :columns="table.columns" :data="table.rows" :mail-action="mailToOwner"
				           :edit-action="editElement" :delete-action="deleteElement" :send-rubrics-action="sendRubrics"></tablegrid>
			</div>
			<div class="col-lg-3">
				<div class="actions well">
					<template v-if="this.$store.getters.user.admin">
						<legend>Actions</legend>
						<ul>
							<li><router-link to="/transmitters/new">New Transmitter</router-link></li>
							<li><p class="linklike" @click="mailToAll">Send a mail to all owners</p></li>
						</ul>
						<br/>
					</template>
					<template v-if="table.rows">
						<legend>Statistics</legend>
						<ul class="list-group">
							<li class="list-group-item"><b>Total Transmitters</b><span class="badge">{{ statTotal }}</span></li>
							<li class="list-group-item"><chart-online-offline :chartData="chartData"></chart-online-offline></li>
							<li class="list-group-item"><chart-transmitter-types :chartData="chartDataDeviceTypes"></chart-transmitter-types></li>
						</ul>
						<div class="checkbox">
							<label><input type="checkbox" v-model="settings.widerangeOnly"> Widerange-transmitter only</label>
						</div>
					</template>
				</div>
			</div>
		</div>
	</div>
</template>

<script>
	import ChartOnlineOffline from '@/components/charts/OnlineOffline';
	import ChartTransmitterTypes from '@/components/charts/TransmitterTypes';

	export default {
		components: {
			ChartOnlineOffline, ChartTransmitterTypes
		},
		created() {
			this.loadData();
		},
		data() {
			return {
				errorMessage: false,
				running: false,
				table: {
					columns: [
						{
							id: 'name',
							title: 'Callsign'
						},
						{
							id: 'nodeName',
							title: 'Node'
						},
						{
							id: 'address',
							title: 'IP-Address'
						},
						{
							id: 'ownerNames',
							title: 'Owner'
						},
						{
							id: 'deviceType',
							title: 'Device'
						},
						{
							id: 'status',
							title: 'Status'
						},
						{
							id: 'connectedSince',
							title: 'Connected since'
						},
						{
							id: 'actions',
							title: 'Actions'
						}
					],
					rows: false
				},
				settings: {
					widerangeOnly: true
				}
			};
		},
		computed: {
			statTotal() {
				if (this.settings.widerangeOnly) {
					return this.table.rows.filter(value => value.usage === 'WIDERANGE').length;
				} else {
					return this.table.rows.length;
				}
			},
			statOnline() {
				if (this.settings.widerangeOnly) {
					return this.table.rows.filter(value => value.status.includes('ONLINE') && value.usage === 'WIDERANGE').length;
				} else {
					return this.table.rows.filter(value => value.status.includes('ONLINE')).length;
				}
			},
			chartData() {
				return {
					labels: ['Online', 'Offline'],
					datasets: [{
						data: [this.statOnline, this.statTotal - this.statOnline],
						backgroundColor: ['#469408', '#D9230F'],
						hoverBackgroundColor: ['#469408', '#D9230F']
					}]
				};
			},
			chartDataDeviceTypes() {
				let chartData = {
					labels: [],
					datasets: [{
						data: [],
						backgroundColor: [],
						hoverBackgroundColor: []
					}]
				};

				if (!this.table.rows || this.table.rows.length === 0) return chartData;

				let returnData = {};
				this.table.rows.forEach(transmitter => {
					// use only transmitters which are currently online
					if (!transmitter.status.includes('ONLINE')) {
						return true;
					}

					// hide non-widerange transmitters if checkbox is set
					if (this.settings.widerangeOnly && transmitter.usage !== 'WIDERANGE') {
						return true;
					}

					let deviceTypeSplitted = transmitter.deviceType.split(' ');
					let deviceType = deviceTypeSplitted[deviceTypeSplitted.length - 1];

					if (!deviceType || deviceType === '---') {
						deviceType = 'Unknown';
					}

					if (returnData[deviceType] !== undefined) {
						returnData[deviceType].amount++;
					} else {
						returnData[deviceType] = {};
						returnData[deviceType].name = deviceType;
						returnData[deviceType].amount = 1;
					}
				});

				let orderedReturnData = {};
				Object.keys(returnData).sort().forEach(key => {
					orderedReturnData[key] = returnData[key];
				});

				const distColors = this.$helpers.getDistributedColors(Object.keys(orderedReturnData).length);
				let count = 0;
				for (let key in orderedReturnData) {
					if (!orderedReturnData.hasOwnProperty(key)) {
						return true;
					}

					chartData.labels.push(orderedReturnData[key].name);
					chartData.datasets[0].data.push(orderedReturnData[key].amount);

					let color = distColors[count];
					chartData.datasets[0].backgroundColor.push(color);
					chartData.datasets[0].hoverBackgroundColor.push(color);

					count++;
				}

				return chartData;
			}
		},
		methods: {
			loadData() {
				this.running = true;
				this.$http.get('transmitters').then(response => {
					// success --> save new data

					response.body.forEach(transmitter => {
						// add ip (if available)
						if (transmitter.address !== undefined && transmitter.address !== null) {
							transmitter.address = transmitter.address.ip_addr;
						} else {
							transmitter.address = '---';
						}

						// add icon to device
						if (transmitter.deviceType === null) transmitter.deviceType = '---';
						if (transmitter.usage === 'WIDERANGE') {
							transmitter.deviceType = '<img src="./assets/img/transmitter_widerange.png" title="Widerange"> ' + transmitter.deviceType;
						} else if (transmitter.usage === 'PERSONAL') {
							transmitter.deviceType = '<img src="./assets/img/transmitter_personal.png" title="Personal"> ' + transmitter.deviceType;
						}

						// make status more colorful
						if (transmitter.status === 'ONLINE') {
							transmitter.status = '<span class="label label-success">ONLINE</span>';
						} else if (transmitter.status === 'OFFLINE') {
							transmitter.status = '<span class="label label-primary">OFFLINE</span>';
						} else {
							transmitter.status = '<span class="label label-warning">' + transmitter.status + '</span>';
						}

						// add actions (if admin or owner)
						transmitter.actions = false;
						if (this.$store.getters.user.admin || transmitter.ownerNames.includes(this.$store.getters.user.name)) {
							transmitter.actions = true;
						}
					});

					this.table.rows = response.body;

					this.running = false;
					this.errorMessage = false;
				}, response => {
					// error --> show error message
					this.running = false;
					this.errorMessage = this.$helpers.getAjaxErrorMessage(response);
				});
			},
			mailToOwner(element) {
				let mailTo = [];
				this.$http.get('users').then(response => {
					response.body.forEach(user => {
						if (element.ownerNames.includes(user.name)) {
							mailTo.push(user.mail);
						}
					});
					window.location.href = 'mailto:' + mailTo.join(',') + '?subject=DAPNET%20Transmitter%3A%20' + element.name;
				}, response => {
					this.$dialogs.ajaxError(this, response);
				});
			},
			editElement(element) {
				this.$router.push({name: 'Edit Transmitter', params: {id: element.name}});
			},
			deleteElement(element) {
				this.$dialogs.deleteElement(this, () => {
					this.$http.delete('transmitters/' + element.name, {
						before(request) {
							request.headers.delete('Content-Type');
						}
					}).then(response => {
						// success --> reload data
						this.loadData();
					}, response => {
						// error --> show error message
						this.$dialogs.ajaxError(this, response);
					});
				});
			},
			sendRubrics(element) {
				this.$http.get('transmitterControl/sendRubricNames/' + element.name).then(() => {
					// success
					this.$dialogs.success(this);
				}, response => {
					// error --> show error message
					this.$dialogs.ajaxError(this, response);
				});
			},
			mailToAll() {
				let mailTo = [];
				this.$http.get('users').then(response => {
					response.body.forEach(user => {
						// check if user owns a transmitter
						this.table.rows.forEach(row => {
							if (row.ownerNames.includes(user.name) && !mailTo.includes(user.mail)) {
								// add user's mail-address if it is not already in the list
								mailTo.push(user.mail);
							}
						});
					});
					window.location.href = 'mailto:' + mailTo.join(',') + '?subject=DAPNET%20Transmitter';
				}, response => {
					this.$dialogs.ajaxError(this, response);
				});
			}
		}
	};
</script>

<style scoped>

</style>
