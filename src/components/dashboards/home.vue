<template>
  <mdb-container fluid>
    <div id="spinner" v-if="showWaiting" role="alert" style="background: transparent; width: 85%; margin-top: 10%; position:absolute;">
      <img alt="waiting" src="/assets/spinner-yokratom.gif">
    </div>

    <section v-if="State === '' && !showWaiting">
        <h2>How to search</h2>
        <mdb-card cascade narrow>
          <mdb-card-body cascade class="pb-0">
            <p style="text-align:left">Select the "<strong>ENTER PATIENT INFO</strong>" button to enter the
              <ul>
                <li>Name and Date of Birth</li>
              </ul>
              <strong>OR</strong>
              <ul>
                <li>Name and Social Security number </li>
              </ul>
            of a VA Patient to search for who accessed their records in the VistA environment
          </p>
          </mdb-card-body>
        </mdb-card>
    </section>

    <section v-if="State === 'Patient Search Complete' && PatientLookupResults && PatientLookupResults.u">
      We have a patient who has {{PatientLookupResults.u[0].length}} users looking at their records
    </section>

    <section class="metadata_table" v-if="State === 'User Selected'">
      <h2>Metadata Records</h2>
      <h3>
        <span class="font-weight-bold">Patient:</span> {{PatientLookupResults.p}}
        <span class="font-weight-bold">User:</span> {{selectedUser.split("-")[0]}}
      </h3>

        <!-- Data Table to display the Metadata info for selected patient/user -->
        <!-- Pass the metadata via the getMetadata return -->
        <data-table
          ref="SelectMetadataRecord"
          v-if=renderDataTableComponent
          :PatientLookupResults="PatientLookupResults"
          :selectedUser="selectedUser"
          :data="getMetadata"
        />
    </section>

    <section id="no_data" v-if="State === 'No Data'">
      <table class="noPatientData">
        <caption>No patients found matching search criteria:</caption>
        <tr><th scope="row">Name: </th><td>{{patientSearchData.name}}</td></tr>
        <tr><th scope="row">Date of Birth: </th><td>{{patientSearchData.dob ? patientSearchData.dob : "Not Entered"}}</td></tr>
        <tr><th scope="row">Social Security Number: </th><td>{{patientSearchData.ssn ? patientSearchData.ssn : "Not Entered"}}</td></tr>
      </table>
    </section>

    <section id="error" v-if="State === 'Error'">
      <div class="alert alert-danger" role="alert">
        Error detected - Network returned {{Error}}
      </div>
    </section>

    <!-- Modal to select the patient -->
    <mdb-select-patient />

    <!-- Modal to display Audit Details -->
    <mdb-show-audit-details :showAuditDetails="showAuditDetails" :auditData="auditData" />

  </mdb-container>
</template>

<script>
import {eventBus} from "../../main";
import { mdbContainer, mdbCard, mdbCardBody } from "mdbvue";
import dataTable from "../components/data_render2";
import mdbSelectPatient from "../components/mdbSelectPatient";
import mdbShowAuditDetails from "../components/mdbAudit";
import axios from "axios";

import vSelect from "vue-select"


export default {
  props: {
    mdHomeTitle: "",
    mdPatientQuery: "",
    mdUserQuery: {},
    mdSelectedPatient: "",
    mdSelectedUser: "",
    PatientLookupResults: {}
  },
  name: "Home",
  components: {
    dataTable,
    mdbSelectPatient,
    mdbShowAuditDetails,
    mdbContainer,
    mdbCard,
    mdbCardBody,
    vSelect
  },

  data() {
    return {
      APIBase4P: process.env.VUE_APP_API_BASE,
      httpHeaders: {
        "Accept": "application/json",
        "Access-Control-Allow-Origin": "*"
      },
      Error: "",
      showWaiting: true,
      initialShowWaiting: true,
      auditDetails: {},
      renderDataTableComponent: false,
      pageTitle: "",
      selectedUser: "",
      patientSearchData: {},
      mdbID: "",
      options: [],
      showAuditDetails: false,
      auditData: {}
    }
  },

  computed: {
    State: function() {
      if (this.pageTitle === "") {
        return "";
      } else if (this.pageTitle === "Start Search") {
        return this.pageTitle;
      } else if (this.pageTitle === "Searching") { // Start searching for specified patient
        return this.pageTitle;
      } else if (this.pageTitle === "Patient Search Complete") {
        return this.pageTitle;
      } else if (this.pageTitle === "Complete") {
        return this.pageTitle;
      } else if (this.pageTitle === "No Data") {
        return this.pageTitle;
      } else if (this.pageTitle === "User Selected") {
        return this.pageTitle;
      } else if (this.pageTitle === "Error") {
        return this.pageTitle;
      }
      console.log("State View Unknown message received ", this.pageTitle);
      return "";
    }
  },

  mounted() {
    let vm = this; // We need the local scope at mount time not at execute time

    // console.log("Mount - Initial Spinner - ", this.showWaiting)

    setTimeout(function() {
      // console.log("Initial Spinner")
      vm.showWaiting = false;
      vm.initialShowWaiting = false;
    }, 2000);


    // Message sent from Select Patient dialog
    // Possible states are : "No Data", "Start Search", "Searching", "Complete"
    eventBus.$on("mdqp-list", async function(message) {
      // console.log("Home.vue received mdqp-list - ", message);
      vm.pageTitle = message.status;
      // vm.hideSpinner();

      // if (message.status !== "Searching") {
      //   vm.hideSpinner();
      // }
      if (message.status === "Error") {
        vm.Error = message.msg;
        vm.hideSpinner();
      } else {
        vm.Error = "";
      }
      if (message.status === "No Data") {
        vm.patientSearchData = message.msg;
        vm.hideSpinner();
      }
      if (message.status === "Patient Search Complete") {
        vm.selectedUser = "";
        vm.hideSpinner();
      }
    });

    eventBus.$on("wait-for-search", async function (state) {
      if (state) {
        vm.showSpinner();
      } else {
        vm.hideSpinner();
      }
    });

    eventBus.$on("SelectedUser", function(theUser) {
      // console.log("Who called SelectedUser? - ", theUser)
      vm.pageTitle = "User Selected";
      vm.selectedUser = theUser;
      vm.forceRerender(false); // Don't render the data table until data is available
    });

    // Event passed when selecting row in data table (data_render2 component), the MetaData record contained in the table is passed.
    eventBus.$on("ShowAuditDetails", async function(mdRecord) {
      // console.log("home.vue - Get and Show Audit Details - mdRecord - ", mdRecord)
      if (mdRecord) {
        const rslt = await vm.getAuditData(mdRecord);
        vm.showAuditDetails = true;

        if (rslt.status === 200) {
          vm.auditData.status = true;
          // OLDBUCK,DUENNA
          // -265243174-19791009
          vm.auditData.auditRecord = rslt.data.response;
          vm.auditData.msg = "";
        } else {
          vm.auditData.status = false;
          vm.auditData.msg = `patient: ${mdRecord.p.split("-")[0]} \nUser: ${mdRecord.u.split("-")[0]}`;
        }
      }
    });

    eventBus.$on("closeAuditRecord", function() {
      if (vm.$refs.SelectMetadataRecord) {
        // console.log("Set focus in Data Table - ", vm.mdbID)
        let theSelected = document.querySelector(".patient-user-metadata tbody td input[value=true]");
        theSelected.focus();
      }
    });

    eventBus.$on("hide", function(ob) { // Then set focus on some element within the modal to allow for "esc" to close it
      // console.log("Hide Audit Record")
      if (ob.showAuditDetails) {
        vm.showAuditDetails = false;
      }
    });
  },

  methods: {
    hideSpinner() {
      let vm = this;
      // console.log("Hide spinner - (delay) ", false, " current - ", this.showWaiting);
      setTimeout(function() {
        vm.showWaiting = false;
        vm.initialShowWaiting = false;
      }, 400);
    },

    showSpinner() {
      // console.log("Show spinner - (never delay) ", true);
      this.showWaiting = true;
    },

    forceRerender(flag) {
      // This is used to force the datatable to render fresh data as soon as it comes in
      // Remove my-component from the DOM
      this.renderDataTableComponent = false;

      if (flag) {
        this.$nextTick(() => {
          // Add the component back in
          this.renderDataTableComponent = true;
        });
      }
    },

    async getAuditData (mdRecord) {
      if (mdRecord) {
        // console.log("We are being passed the audit details - ", mdRecord)
        let vm = this;
        vm.mdbID = mdRecord.mdbID;
        let url = `${this.APIBase4P}auditById?id=${mdRecord._id}`;
        // console.log("getAuditData Searching; URL: ", url)
        vm.showSpinner();
        let rslt = {};
        try {
          rslt = await axios.get(url, { headers: this.httpHeaders });
          vm.hideSpinner();
          vm.auditDetails = rslt.data.response;
          // debugger;
          // console.log("Result - ", rslt);
          // Fake this for now till we can update the metadata to return a 204
          if (rslt.data.response === "No Matching Records found") {
            rslt.status = 204;
          }
          return rslt;
        } catch (err) {
          const errState = `initialConnection - ${url} Query failed - ${JSON.stringify(err)}`;
          // console.log(errState);
          rslt.status = 204;
          rslt.statusText = errState;
          rslt.data.response = errState;
          return rslt;
        }
      }
    }
  },

  asyncComputed: {
    getMetadata: {
      async get () {
        if (this.PatientLookupResults && this.selectedUser) {
          // console.log("getMetadata in home.vue - starting lookup of Metadata")
          let vm = this;
          const columns = [
            { label: "Field Changed", field: "c", sort: true },
            {
              label: "Date",
              field: "d",
              sort: true,
              format: function(el) {
                let y = el.substr(0, 4);
                let m = el.substr(4, 2);
                let d = el.substr(6, 2);
                let h = el.substr(8, 2);
                let mi = el.substr(10, 2);
                let s = el.substr(12, 2);
                let z = el.substr(15, 4);
                return `${m}/${d}/${y} - ${h}:${mi}:${s} - ${z}Z`;
              }
            },
            {
              label: "Site data accessed from", field: "l", sort: true
            }
          ]
          let p = this.PatientLookupResults.key;
          let u = this.selectedUser
          let url = `${this.APIBase4P}meta?p=${p}&u=${u}`;

          let theResponse = "";
          // console.log("home.vue - mdqp-list - Searching - getMetadata ", url)
          eventBus.$emit("wait-for-search", true);
          await axios.get(url, { headers: this.httpHeaders })
            .then(function(rslt) {
              theResponse = { columns: columns, rows: rslt.data };
              vm.hideSpinner();
            })
            .catch(function(e) {
              // console.log(`initialConnection - ${url} Query failed - `, e);
              theResponse = "A Failure - " + e;
              // eventBus.$emit("mdqp-list", { status: "Error", msg: theResponse });
              vm.pageTitle = "Error";
              // vm.State = "Error";
              // vm.Error = theResponse;
              vm.hideSpinner();
            })
          vm.forceRerender(true);
          return theResponse;
        }
        // else {
        // console.log("getMetadata in home.vue - is NOT looking for data - ")
        // }
      }
    }
  }
}
</script>

<!-- Add 'scoped' attribute to limit CSS to this component only -->
<style scoped>

.metadata_table, .metadata_table div.container {
  max-width:100%;
}

.metadata_table h3 {
  font-size: 1.0rem;
}

.noPatientData table {
  margin-left: 20%
}

.noPatientData caption {
  caption-side:top;
  text-align: center;
}

.noPatientData th {
  font-weight: bold;
  text-align: right;
  width:50%;
  margin-left: 25%;
}
.noPatientData td {
  text-align: left;
}

h2 {
  font-weight: bold;
  text-align: center;
}

.container-fluid {
  height: 80vh;
}
.card {
  height: 70vh;
}

</style>
