<template>
<div>
	<q-drawer side="left" :value="true" :width="400">
		<q-splitter horizontal v-model="splitterRatio" :limits="[50, 80]" style="height: 100%">
			<template v-slot:before>
				<q-list class="home-drawer">
					<q-item style="padding:0px">
						<q-item-section class="q-mx-md">{{$t('sections')}}</q-item-section>
						<template v-if="$settings.reviews.enabled">
						<q-item-section side class="topButtonSection" v-if="frontEndAuditState === AUDIT_VIEW_STATE.EDIT">
							<q-btn class="q-mx-xs q-px-xs" size="11px" unelevated dense color="secondary" :label="$t('btn.topButtonSection.submitReview')" no-caps @click="toggleAskReview" >
								<q-tooltip anchor="bottom middle" self="center left" :delay="500" content-class="text-bold">{{$t('tooltip.topButtonSection.submitReview')}}</q-tooltip> 
							</q-btn>
						</q-item-section>
						<q-item-section side class="topButtonSection" v-if="[AUDIT_VIEW_STATE.REVIEW_EDITOR, AUDIT_VIEW_STATE.REVIEW_ADMIN, AUDIT_VIEW_STATE.REVIEW_ADMIN_APPROVED].includes(frontEndAuditState)">
							<q-btn class="q-mx-xs q-px-xs" size="11px" unelevated dense color="amber-9" :label="$t('btn.topButtonSection.cancelReview')" no-caps @click="toggleAskReview" >
								<q-tooltip anchor="bottom middle" self="center left" :delay="500" content-class="text-bold">{{$t('tooltip.topButtonSection.cancelReview')}}</q-tooltip> 
							</q-btn>
						</q-item-section>
						<q-item-section side class="topButtonSection" v-if="[AUDIT_VIEW_STATE.REVIEW, AUDIT_VIEW_STATE.REVIEW_ADMIN].includes(frontEndAuditState)">
							<q-btn class="q-mx-xs q-px-xs" size="11px" unelevated dense color="green" :label="$t('btn.topButtonSection.approve')" no-caps @click="toggleApproval" >
								<q-tooltip anchor="bottom middle" self="center left" :delay="500" content-class="text-bold">{{$t('tooltip.topButtonSection.approve')}}</q-tooltip> 
							</q-btn>
						</q-item-section>
						<q-item-section side class="topButtonSection" v-if="[AUDIT_VIEW_STATE.REVIEW_APPROVED, AUDIT_VIEW_STATE.REVIEW_ADMIN_APPROVED, AUDIT_VIEW_STATE.APPROVED_APPROVED].includes(frontEndAuditState)">
							<q-btn class="q-mx-xs q-px-xs" size="11px" unelevated dense color="warning" :label="$t('btn.topButtonSection.removeApproval')" no-caps @click="toggleApproval" >
								<q-tooltip anchor="bottom middle" self="center left" :delay="500" content-class="text-bold">{{$t('tooltip.topButtonSection.removeApproval')}}</q-tooltip> 
							</q-btn>
						</q-item-section>
						</template>
						<q-item-section side class="topButtonSection">
							<q-btn flat color="info" @click="generateReport">
								<q-tooltip anchor="bottom middle" self="center left" :delay="500" content-class="text-bold">{{$t('tooltip.downloadReport')}}</q-tooltip> 
								<i class="fa fa-download fa-lg"></i>
							</q-btn>
						</q-item-section>
					</q-item>

					<q-item :to='"/audits/"+auditId+"/general"'>
						<q-item-section avatar>
							<q-icon name="fa fa-cog"></q-icon>
						</q-item-section>
						<q-item-section>{{$t('generalInformation')}}</q-item-section>
					</q-item>
					
					<div class="row">
						<div v-for="(user,idx) in generalUsers" :key="idx" class="col multi-colors-bar" :style="{background:user.color}" />
					</div>

					<q-item
					v-if="!currentAuditType || !currentAuditType.hidden.includes('network')"
					:to="'/audits/'+auditId+'/network'"
					>
						<q-item-section avatar>
							<q-icon name="fa fa-globe"></q-icon>
						</q-item-section>
						<q-item-section>{{$t('networkScan')}}</q-item-section>
					</q-item>

					<div class="row">
						<div v-for="(user,idx) in networkUsers" :key="idx" class="col multi-colors-bar" :style="{background:user.color}" />
					</div>

					<div v-if="!currentAuditType || !currentAuditType.hidden.includes('findings')">
						<q-separator class="q-my-sm" />
						<q-item>
							<q-item-section avatar>
								<q-icon name="fa fa-list"></q-icon>
							</q-item-section>
							<q-item-section>{{$t('findings')}} ({{audit.findings.length || 0}})</q-item-section>
							<q-item-section avatar>
								<q-btn
								@click="$router.push('/audits/'+auditId+'/findings/add').catch(err=>{})"
								icon="add"
								round
								dense
								color="secondary"
								v-if="frontEndAuditState === AUDIT_VIEW_STATE.EDIT"
								/>
							</q-item-section>
						</q-item>
						
						<div v-for="categoryFindings of findingList" :key="categoryFindings.category">
							<q-item>
								<q-item-section>
									<q-item-label header>{{categoryFindings.category}}</q-item-label>
								</q-item-section>
								<q-item-section avatar>
									<q-btn icon="sort" flat v-if="frontEndAuditState === AUDIT_VIEW_STATE.EDIT">
										<q-tooltip anchor="bottom middle" self="center left" :delay="500" content-class="text-bold">{{$t('tooltip.sortOptions')}}</q-tooltip>
										<q-menu content-style="width: 300px" anchor="bottom middle" self="top left" content-class="bg-grey-1">
											<q-item>
												<q-item-section>
													<q-toggle 
													v-model="categoryFindings.sortOption.sortAuto" 
													:label="$t('automaticSorting')"
													@input="updateSortFindings"
													/>
												</q-item-section>
											</q-item>
											<q-separator />
											<q-item>
												<q-item-section>
													<q-item-label>{{$t('sortBy')}}</q-item-label>
												</q-item-section>
											</q-item>
											<q-item>
												<q-item-section>
													<q-option-group
													v-model="categoryFindings.sortOption.sortValue"
													:options="getSortOptions(categoryFindings.sortOption.category)"
													type="radio"
													:disable="!categoryFindings.sortOption.sortAuto"
													@input="updateSortFindings"
													/>
												</q-item-section>
											</q-item>
											<q-separator />
											<q-item>
												<q-item-section>
													<q-btn 
													flat
													icon="fa fa-long-arrow-alt-up"
													:label="$t('ascending')"
													dense
													no-caps
													align="left"
													:disable="!categoryFindings.sortOption.sortAuto"
													:color="(categoryFindings.sortOption.sortOrder === 'asc')?'green':'black'" 
													@click="categoryFindings.sortOption.sortOrder = 'asc'; updateSortFindings()" 
													/>
												</q-item-section>
											</q-item>
											<q-item>
												<q-item-section>
													<q-btn 
													flat
													icon="fa fa-long-arrow-alt-down"
													:label="$t('descending')"
													dense
													no-caps
													align="left"
													:disable="!categoryFindings.sortOption.sortAuto"
													:color="(categoryFindings.sortOption.sortOrder === 'desc')?'green':'black'" 
													@click="categoryFindings.sortOption.sortOrder = 'desc'; updateSortFindings()" 
													/>
												</q-item-section>
											</q-item>
										</q-menu>
									</q-btn>
								</q-item-section>
							</q-item>
							<q-list no-border>
								<draggable :list="categoryFindings.findings" @end="moveFindingPosition($event, categoryFindings.category)" handle=".handle" ghost-class="drag-ghost">
									<div v-for="finding of categoryFindings.findings" :key="finding._id">
										<q-item
										dense
										class="cursor-pointer"
										:to="'/audits/'+auditId+'/findings/'+finding._id"
										>
											<q-item-section side v-if="!categoryFindings.sortOption.sortAuto && frontEndAuditState === AUDIT_VIEW_STATE.EDIT">
												<q-icon name="mdi-arrow-split-horizontal" class="cursor-pointer handle" color="grey" />
											</q-item-section>
											<q-item-section side>
												<q-chip
													class="text-white"
													size="sm"
													square
													:style="`background: ${getFindingColor(finding)}`"
												>{{getFindingSeverity(finding).substring(0,1)}}</q-chip>
											</q-item-section>
											<q-item-section>
												<span>{{finding.title}}</span>
											</q-item-section>
											<q-item-section side v-if="finding.status === 0">
												<q-icon name="check" color="green" />
											</q-item-section>
										</q-item>
										<div class="row">
											<div v-for="(user,idx) in findingUsers" :key="idx" v-if="user.finding === finding._id" class="col multi-colors-bar" :style="{background:user.color}" />
										</div>
									</div>
								</draggable>
							</q-list>
						</div>
						<q-separator class="q-my-sm" />
					</div>
					<q-list v-for="section of audit.sections" :key="section._id">
						<q-item :to="'/audits/'+auditId+'/sections/'+section._id">
							<q-item-section avatar>
								<q-icon :name="getSectionIcon(section)"></q-icon>
							</q-item-section>
							<q-item-section>
								<span>{{section.name}}</span>
							</q-item-section>
						</q-item>
						<div class="row">
							<div v-for="(user,idx) in sectionUsers" :key="idx" v-if="user.section === section._id" class="col multi-colors-bar" :style="{background:user.color}" />
						</div>
					</q-list>
				</q-list>
			</template>
			<template v-slot:after>
				<q-list>
					<q-separator />
					<q-item class="q-py-lg">
						<q-item-section avatar>
							<q-icon name="fa fa-user"></q-icon>
						</q-item-section>
						<q-item-section>{{$t('usersConnected')}}</q-item-section>	
					</q-item>
					<q-list dense>
						<q-item v-for="user of users" :key="user._id">
							<q-item-section side>
								<q-chip :style="{'background-color':user.color}" square size="sm" />
							</q-item-section>
							<q-item-section>
								<span v-if="user.me">{{user.username}} ({{$t('me')}})</span>
								<span v-else>{{user.username}}</span>
							</q-item-section>
						</q-item>
					</q-list>
				</q-list>
			</template>
			
		</q-splitter>
	</q-drawer>
	<router-view :key="$route.fullPath" :frontEndAuditState="frontEndAuditState" :parentState="audit.state" :parentApprovals="audit.approvals" />
	</div>
</template>

<script>
import { Notify, QSpinnerGears } from 'quasar';
import draggable from 'vuedraggable'

import AuditService from '@/services/audit';
import UserService from '@/services/user';
import DataService from '@/services/data';
import Utils from '@/services/utils';

import { $t } from '@/boot/i18n';

export default {
		data () {
				return {
					auditId: "",
					findings: [],
					users: [],
					audit: {findings: {}},
					sections: [],
					splitterRatio: 80,
					loading: true,
					vulnCategories: [],
					customFields: [],
					auditTypes: [],
					vulnCategories: [],
					findingList: [],
					frontEndAuditState: Utils.AUDIT_VIEW_STATE.EDIT_READONLY,
					AUDIT_VIEW_STATE: Utils.AUDIT_VIEW_STATE
				}
		},

		components: {
			draggable
		},

		created: function() {
			this.auditId = this.$route.params.auditId;
			this.getCustomFields();
			this.getAuditTypes();
			this.getAudit(); // Calls getSections				
		},

		destroyed: function() {
			if (!this.loading) {
				this.$socket.emit('leave', {username: UserService.user.username, room: this.auditId});
				this.$socket.off()
			}
		},

		watch: {
			'audit.findings': {
				handler(newVal, oldVal) {
					var result = _.chain(this.audit.findings)
					.groupBy("category")
					.map((value, key) => {
						if (key === 'undefined') key = 'No Category'
						var sortOption = this.audit.sortFindings.find(option => option.category === key) // Get sort option saved in audit
						
						if (!sortOption) { // no option for category in audit
							sortOption = this.vulnCategories.find(e => e.name === key) // Get sort option from default in vulnerability category
							if (sortOption) // found sort option from vuln categories
								sortOption.category = sortOption.name
							else // no default option or category don't exist
								sortOption = {category: key, sortValue: 'cvssScore', sortOrder: 'desc', sortAuto: true} // set a default sort option
							
							this.audit.sortFindings.push({
								category: sortOption.category,
								sortValue: sortOption.sortValue,
								sortOrder: sortOption.sortOrder,
								sortAuto: sortOption.sortAuto
							})
						}
						
						return {category: key, findings: value, sortOption: sortOption}
					})
					.value()

					this.findingList = result
				},
				deep: true,
				immediate: true
			}
		},

		computed: {
			generalUsers: function() {return this.users.filter(user => user.menu === 'general')},
			networkUsers: function() {return this.users.filter(user => user.menu === 'network')},
			findingUsers: function() {return this.users.filter(user => user.menu === 'editFinding')},
			sectionUsers: function() {return this.users.filter(user => user.menu === 'editSection')},

			currentAuditType: function() {
				return this.auditTypes.find(e => e.name === this.audit.auditType)
			}
		},

		methods: {
			getFindingColor: function(finding) {
				let severity = this.getFindingSeverity(finding)

				if(this.$settings.report) {
					const severityColorName = `${severity.toLowerCase()}Color`;
					const cvssColors = this.$settings.report.public.cvssColors;

					return cvssColors[severityColorName] || cvssColors.noneColor;
				} else {
					switch(severity) {
						case "Low": 
							return "green";
						case "Medium":
							return "orange";
						case "High":
							return "red";
						case "Critical":
							return "black";
						default:
							return "blue";
					}
				}
			},

			getFindingSeverity: function(finding) {
				let severity = "None"
				let cvss = CVSS31.calculateCVSSFromVector(finding.cvssv3)
				if (cvss.success)
					severity = cvss.baseSeverity
				return severity
			},

			// Sockets handle
			handleSocket: function() {
				this.$socket.emit('join', {username: UserService.user.username, room: this.auditId});
				this.$socket.on('roomUsers', (users) => {
					var userIndex = 0;
					this.users = users.map((user,index) => {
						if (user.username === UserService.user.username) {
							user.color = "#77C84E";
							user.me = true;
							userIndex = index;
						}
						return user;
					});
					this.users.unshift(this.users.splice(userIndex, 1)[0]);
				})
				this.$socket.on('updateUsers', () => {
					this.$socket.emit('updateUsers', {room: this.auditId})
				})
				this.$socket.on('updateAudit', () => {
					this.getAudit();
				})
			},
			// Tells the UI if the user is supposed to be reviewing the audit
			isUserAReviewer: function() {
				var isAuthor = this.audit.creator._id === UserService.user.id;
				var isCollaborator = this.audit.collaborators.some((element) => element._id === UserService.user.id);
				var isReviewer = this.audit.reviewers.some((element) => element._id === UserService.user.id);
				var hasReviewAll = UserService.isAllowed('audits:review-all');
				return !(isAuthor || isCollaborator) && (isReviewer || hasReviewAll);
			},

			// Tells the UI if the user is supposed to be editing the audit
			isUserAnEditor: function() {
				var isAuthor = this.audit.creator._id === UserService.user.id;
				var isCollaborator = this.audit.collaborators.some((element) => element._id === UserService.user.id);
				var hasUpdateAll = UserService.isAllowed('audits:update-all');
				return (isAuthor || isCollaborator || hasUpdateAll);
			},

			userHasAlreadyApproved: function() {
				return this.audit.approvals.some((element) => element._id === UserService.user.id);
			},

			getUIState: function() {
				if(!this.$settings.reviews.enabled || this.audit.state === "EDIT") {
					this.frontEndAuditState = this.isUserAnEditor() ? Utils.AUDIT_VIEW_STATE.EDIT : Utils.AUDIT_VIEW_STATE.EDIT_READONLY;
				} 
				else if (this.audit.state === "REVIEW") {
					if (!this.isUserAReviewer()) {
						this.frontEndAuditState = this.isUserAnEditor()? Utils.AUDIT_VIEW_STATE.REVIEW_EDITOR : Utils.AUDIT_VIEW_STATE.REVIEW_READONLY;
						return;
					}
					if (this.isUserAnEditor()) {
						this.frontEndAuditState = this.userHasAlreadyApproved() ? Utils.AUDIT_VIEW_STATE.REVIEW_ADMIN_APPROVED : Utils.AUDIT_VIEW_STATE.REVIEW_ADMIN;
						return;
					}
					this.frontEndAuditState = this.userHasAlreadyApproved() ? Utils.AUDIT_VIEW_STATE.REVIEW_APPROVED : Utils.AUDIT_VIEW_STATE.REVIEW;
				} 
				else if (this.audit.state === "APPROVED") {
					if (!this.isUserAReviewer()) {
						this.frontEndAuditState = Utils.AUDIT_VIEW_STATE.APPROVED_READONLY;
					} else {
						this.frontEndAuditState = this.userHasAlreadyApproved() ? Utils.AUDIT_VIEW_STATE.APPROVED_APPROVED : Utils.AUDIT_VIEW_STATE.APPROVED
					}
				}
			},

			getAudit: function() {
				DataService.getVulnerabilityCategories() // Vuln Categories must exist before getting audit data for handling default sort options
				.then(data => {
					this.vulnCategories = data.data.datas
					return AuditService.getAudit(this.auditId)
				})
				.then((data) => {
					this.audit = data.data.datas;
					this.getUIState();
					this.getSections()
					if (this.loading)
						this.handleSocket()
					this.loading = false
				})
				.catch((err) => {
					if (err.response.status === 403)
						this.$router.push({name: '403', params: {error: err.response.data.datas}})
					else if (err.response.status === 404)
						this.$router.push({name: '404', params: {error: err.response.data.datas}})
				})
			},

			getCustomFields: function() {
				DataService.getCustomFields()
				.then((data) => {
					this.customFields = data.data.datas;
				})
				.catch((err) => {
					console.log(err);
				})
			},

			getSections: function() {
				DataService.getSections()
				.then((data) => {
					this.sections = data.data.datas;
				})
				.catch((err) => {
					console.log(err);
				})
			},

			getSectionIcon: function(section) {
				var section = this.sections.find(e => e.field === section.field)
				if (section)
					return section.icon || 'notes'
				return 'notes'
			},

			getAuditTypes: function() {
				DataService.getAuditTypes()
				.then((data) => {
					this.auditTypes = data.data.datas;
				})
				.catch((err) => {
					console.log(err);
				})
			},

			// Convert blob to text
			BlobReader: function(data) {
				const fileReader = new FileReader();

				return new Promise((resolve, reject) => {
					fileReader.onerror = () => {
						fileReader.abort()
						reject(new Error('Problem parsing blob'));
					}

					fileReader.onload = () => {
						resolve(fileReader.result)
					}

					fileReader.readAsText(data)
				})
			},

			generateReport: function() {
				const downloadNotif = Notify.create({
					spinner: QSpinnerGears,
					message: 'Generating the Report',
					color: "blue",
					timeout: 0,
					group: false
				})
				AuditService.generateAuditReport(this.auditId)
				.then(response => {
					var blob = new Blob([response.data], {type: "application/octet-stream"});
					var link = document.createElement('a');
					link.href = window.URL.createObjectURL(blob);
					link.download = response.headers['content-disposition'].split('"')[1];
					document.body.appendChild(link);
					link.click();
					link.remove();
					
					downloadNotif({
						icon: 'done',
						spinner: false,
						message: 'Report successfully generated',
						color: 'green',
						timeout: 2500
					})
				})
				.catch( async err => {
					var message = "Error generating template"
					if (err.response && err.response.data) {
						var blob = new Blob([err.response.data], {type: "application/json"})
						var blobData = await this.BlobReader(blob)
						message = JSON.parse(blobData).datas
					}
					downloadNotif()
					Notify.create({
						message: message,
						type: 'negative',
						textColor:'white',
						position: 'top',
						closeBtn: true,
						timeout: 0,
						classes: "text-pre-wrap"
					})
				})
			},

			updateSortFindings: function() {
				AuditService.updateAuditSortFindings(this.auditId, {sortFindings: this.audit.sortFindings})
			},

			getSortOptions: function(category) {
				var options = [
					{label: $t('cvssScore'), value: 'cvssScore'},
					{label: $t('priority'), value: 'priority'},
					{label: $t('remediationDifficulty'), value: 'remediationComplexity'}
				]
				var allowedFieldTypes = ['date', 'input', 'radio', 'select']
				this.customFields.forEach(e => {
					if (
						(e.display === 'finding' || e.display === 'vulnerability') && 
						(!e.displaySub || e.displaySub === category) && 
						allowedFieldTypes.includes(e.fieldType)
					) {
						options.push({label: e.label, value: e.label})
					}
				})
				return options
			},

			moveFindingPosition: function(event, category) {
				var index = this.audit.findings.findIndex(e => {
					if (category === 'No Category')
						return !e.category
					else
						return e.category === category
				})
				if (index > -1) {
					var realOldIndex = event.oldIndex + index
					var realNewIndex = event.newIndex + index

					AuditService.updateAuditFindingPosition(this.auditId, {oldIndex: realOldIndex, newIndex: realNewIndex})
					.then(msg => this.getAudit())
					.catch(err => {
						console.log(err.response.data.datas)
						Notify.create({
							message: err.response.data.datas || err.message,
							color: 'negative',
							textColor:'white',
							position: 'top-right'
						})
						this.getAudit()
					})
				}
			},

			toggleAskReview: function() {
				AuditService.updateReadyForReview(this.auditId, { state: this.audit.state === "EDIT" ? "REVIEW" : "EDIT" })
				.then(() => {
					this.audit.state = this.audit.state === "EDIT" ? "REVIEW" : "EDIT";
					this.getUIState();
					Notify.create({
						message: $t('msg.auditReviewUpdateOk'),
						color: 'positive',
						textColor:'white',
						position: 'top-right'
					})
				})
				.catch((err) => {     
					console.log(err)
					Notify.create({
						message: err.response.data.datas || err.message,
						color: 'negative',
						textColor:'white',
						position: 'top-right'
					})
				});
			},

			toggleApproval: function() {
				AuditService.toggleApproval(this.auditId)
				.then(() => {
					Notify.create({
						message: $t('msg.auditApprovalUpdateOk'),
						color: 'positive',
						textColor:'white',
						position: 'top-right'
					})
				})
				.catch((err) => {          
					console.log(err)
					Notify.create({
						message: err.response.data.datas || err.message,
						color: 'negative',
						textColor:'white',
						position: 'top-right'
					})
				});
			}
		}
}
</script>

<style lang="stylus">
.edit-container {
		margin-top: 50px;
		/*margin-left: 0px; Cancel q-col-gutter-md for left*/
		/*margin-right: 16px; Cancel q-col-gutter-md for right*/
}

.edit-breadcrumb {
		position: fixed;
		top: 50px;
		right: 0;
		left: 300px;
		z-index: 1;
}

.q-menu > .q-item--active {
		color: white;
		background-color: $blue-14;
}

.card-screenshots {
	height: calc(100vh - 120px); /* 100% Full-height */
	overflow-x: hidden; /* Disable horizontal scroll */
	margin-right: 16px;
}

.affix {
	width: calc(16.6667% - 69px);
}

.caption-text input {
		text-align: center;
}

.multi-colors-bar {
	height: 5px;
}

.drawer-footer {
	// left: 0!important;
	// height: 30%;
	background-color: white;
	color: black;
	font-size: 12px;
}

.edit-drawer {
	// height: 70%;

}

.topButtonSection {
    padding-left: 0px!important;
	padding-right: 0px!important;
}
</style>