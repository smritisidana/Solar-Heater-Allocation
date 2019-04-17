# Solar-Heater-Allocation
#routing.js
const express = require('express');
const routing = express.Router();
const allocateService = require('../service/allocate')
const Customer = require('../model/customer')

routing.get('/evaluate', (req, res, next) => {
    console.log("Request came!!");
    allocateService.evaluate()
        .then((data) => {
            console.log("Completed success!!");
            res.send(data)
        }).catch((err) => {
            console.log("Completed err!!");
            next(err)
        })
})

routing.put('/allocate/:distributor', (req, res, next) => {
    let assign = new Customer(req.body);
    allocateService.allocate(req.params.distributor, assign).then((custId) => {
        res.json({ "message": "Solar Heater " + req.params.distributor + " successfully allocated to customer " + custId })
    }).catch(function (err) {
        next(err);
    })
})

routing.get('/findService/:location', (req, res, next) => {
    allocateService.fetchDetails(req.params.location).then((data) => {
        res.send(data)
    }).catch((err) => {
        next(err);
    })
})




module.exports = routing;


#errorlogger.js
var fs = require('fs');

var errorLogger = (err, req, res, next) => {
    if (err) {
        fs.appendFile('ErrorLogger.txt', new Date() + " - " + err.stack + "\n", function (error) {
            if (error) {
                console.log("Failed in logging error");
            }
        });

        if (err.status) {
            res.status(err.status)
        }
        else {
            res.status(500);
        }
        res.json({ "message": err.message })
    }
    next();
}

module.exports = errorLogger;

#requestlogger.js
var fs = require('fs');

var requestLogger = (req, res, next) => {
    var logMessage = "" + new Date() + " " + req.method + " " + req.url + "\n";
    fs.appendFile('RequestLogger.txt', logMessage, (err) => {
        if (err) return next(err);
    });
    next();
}

module.exports = requestLogger;

#app.js
const express = require('express');
const bodyParser = require('body-parser');
const router = require('./routes/routing');
const myErrorLogger = require('./utilities/errorlogger');
const myRequestLogger = require('./utilities/requestlogger');
const create = require("./model/dbsetup")
const cors = require('cors');
const app = express();

app.use(cors());
app.use(bodyParser.json());
app.use(myRequestLogger);
app.use('/', router);
app.use(myErrorLogger);

app.get('/setupDb', (req, res, next) => {
    create.setupDb().then((data) => {
        res.send(data)
    }).catch((err) => {
        next(err)
    })
})

app.listen(2040);
console.log("Server listening in port 2040");

module.exports = app;

#GetDistributor.js
import React, { Component } from "react";
import axios from "axios";
import "../App.css";

const url = "http://localhost:2040/findService/";

class GetDistributors extends Component {
  constructor(props) {
    super(props);
    this.state = {
      distributorData: [],
      form: {
        customerLocation: ""
      },
      formErrorMessage: {
        customerLocation: ""
      },
      formValid: {
        customerLocation: false,
        buttonActive: false
      },
      errorMessage: "",
      successMessage: ""
    };
  }

  handleSubmit = (event) => {
    event.preventDefault();
    this.fetchDistributorByLocation();
    /* prevent page reload and invoke fetchDistributorByLocation() method */
  }

  handleChange = (event) => {
    const target = event.target;
    const value = target.value;
    const name = target.name;
    const { form } = this.state;
    this.setState({form: { ...form, [name]: value }});
    this.validateField(name, value);
    /* 
      invoke whenever any change happens in any of the input fields
      and update form state with the value. Also, Inoke validateField() method to validate the entered value
    */
  }

  validateField = (fieldName, value) => {
    if(fieldName == "customerLocation"){
      var locationValid = false;
      if(value.match(/^[A-z][A-z\s]{2,}$/)){
        locationValid = true;
        this.setState(Object.assign(this.state.formErrorMessage,{customerLocation : ""}));
      }
      else if(value == ""){
        this.setState(Object.assign(this.state.formErrorMessage,{customerLocation: <error className="text-danger">field required</error>}));
      }
      else{
        this.setState(Object.assign(this.state.formErrorMessage,{customerLocation:<error className="text-danger">Please enter valid Location</error>}));
      }
      this.setState(Object.assign(this.state.formValid,{customerLocation : locationValid}));
      this.setState(Object.assign(this.state.formValid,{buttonActive : locationValid}));
      this.setState({ distributorData: "", errorMessage: "" })
    }
    /* Perform Validations and assign error messages, Also, set the value of buttonActive after validation of the field */
  }

  fetchDistributorByLocation = () => {
    this.setState({ distributorData: "", errorMessage: "" })
    axios.get(url + this.state.form.customerLocation)
    .then(response => {
      this.setState({ distributorData: response.data, errorMessage: "" });
      var arr = response.data;
      var arr1 = arr.map(item =>{
        return(
          <li>{item}</li>
        )
      })
      this.setState({successMessage : <div> The distributors in {this.state.form.customerLocation} are: <ol> {arr1} </ol></div>});
      // console.log(response.data)
    }).catch(error => {
      if(error.response == null){
        this.setState({ errorMessage: "Server Error", successMessage: "" });
      }
      else{
        this.setState({ errorMessage: error.response.data.message, successMessage: "" });
      }
      });
    /* 
      Send an AXIOS GET request to the url http://localhost:1050/findService/:customerLocation 
      to fetch all the distributors available in that location
      and handle the success and error cases appropriately 
    */
  }

  render() {
    return (
      <React.Fragment>
        <div className="row">
          <div className="col-md-6 offset-md-3">
            <br />
            <div className="card">
              <div className="card-header bg-custom text-center">
                <h4>View Distributors</h4>
              </div>
              <div className="card-body view">
              <form>
              <div className="form-group">
                <label>Customer Location</label>
                <input name="customerLocation" className="form-control" type="text" onChange={this.handleChange} required/>
                <span name="customerLocationError"> {this.state.formErrorMessage.customerLocation}</span>
              </div>
              <button name="getDistributors" type="button" className="btn btn-primary" disabled={!this.state.formValid.buttonActive} onClick={this.handleSubmit}>Get Distributors</button><br/><br/>
              <span name="errorMessage" className="text-danger text-bold">{this.state.errorMessage}</span>
              <span name="successMessage" className="text-success text-bold">{this.state.successMessage}</span>
              </form>
                {/* code here to get the view as shown in QP for GetDistributors component */}
                {/* display the distributors in an unordered list */}
                {/* Display error message if the server is not running */}
              </div>
            </div>
          </div>
        </div>
      </React.Fragment>
    );
  }
}

export default GetDistributors;

#AllocateSolar.js
import React, { Component } from "react";
import axios from "axios";
import "../App";

const url = "http://localhost:2040/allocate/";

class AllocateSolar extends Component {
  constructor(props) {
    super(props);
    this.state = {
      formValue: {
        distributorName: "",
        purchaseDate: "",
        installationDate: "",
        customerName: "",
        customerLocation: ""
      },
      formErrorMessage: {
        distributorName: "",
        purchaseDate: "",
        installationDate: "",
        customerName: "",
        customerLocation: ""
      },
      formValid: {
        distributorName: false,
        purchaseDate: false,
        installationDate: false,
        customerName: false,
        customerLocation: false,
        buttonActive: false
      },
      successMessage: "",
      errorMessage: ""
    }
  }

  submitAllocation = () => {
    const { formValue } = this.state;
    this.setState({ errorMessage: "", successMessage: "" })
    axios.put(url + this.state.formValue.distributorName, formValue)
      .then(response => {
        this.setState({ successMessage: response.data.message, errorMessage: "" });
      }).catch(error => {
        if(error.response == null){
          this.setState({ errorMessage: "Server Error", successMessage: "" });
        }
        else{
          this.setState({ errorMessage: error.response.data.message, successMessage: "" });
        }
      });
    /* 
      Make aN axios PUT request to http://localhost:2040/allocate/:distributorName with form data 
      and handle success and error cases 
    */
  }

  handleSubmit = (event) => {
    event.preventDefault();
    this.submitAllocation();
    /* prevent page reload and invoke submitAllocation() method */
  }

  handleChange = (event) => {
    const target = event.target;
    const value = target.value;
    const name = target.name;
    const { formValue } = this.state;
    this.setState({ formValue: { ...formValue, [name]: value } });
    this.validateField(name, value);
    /* 
      invoke whenever any change happens in any of the input fields
      and update form state with the value. Also, Inoke validateField() method to validate the entered value
    */
  }

  validateField = (fieldName, value) => {
    if(fieldName == "customerId"){
      var distributorValid = false;
      if(value == ""){
        this.setState(Object.assign(this.state.formErrorMessage,{distributorName : <error className="text-danger">Please select distributor name</error>}))
      }
      else{
        distributorValid = true;
        this.setState(Object.assign(this.state.formErrorMessage,{distributorName : ""}))
      }
      this.setState(Object.assign(this.state.formValid,{distributorName : distributorValid}));
      this.setState(Object.assign(this.state.formValid,{buttonActive : distributorValid && this.state.formValid.customerName && this.state.formValid.customerLocation && this.state.purchaseDate && this.state.installationDate}));
    }

    else if(fieldName == "customerName"){
      var customerValid = false;
      if(value.match(/^[A-z][A-z\s]{3,}$/)){
        customerValid = true;
        this.setState(Object.assign(this.state.formErrorMessage,{customerName: ""}));
      }
      else if(value == ""){
        this.setState(Object.assign(this.state.formErrorMessage,{customerName: <error className="text-danger">field required</error>}));
      }
      else{
        this.setState(Object.assign(this.state.formErrorMessage,{customerName:<error className="text-danger">Please enter valid Name</error>}));
      }
      this.setState(Object.assign(this.state.formValid,{customerName : customerValid}));
      this.setState(Object.assign(this.state.formValid,{buttonActive : this.state.distributorName && customerValid && this.state.formValid.customerLocation && this.state.purchaseDate && this.state.installationDate}));
    }

    else if(fieldName == "customerLocation"){
      var locationValid = false;
      if(value.match(/^[A-z][A-z\s]{2,}$/)){
        locationValid = true;
        this.setState(Object.assign(this.state.formErrorMessage,{customerLocation : ""}));
      }
      else if(value == ""){
        this.setState(Object.assign(this.state.formErrorMessage,{customerLocation: <error className="text-danger">field required</error>}));
      }
      else{
        this.setState(Object.assign(this.state.formErrorMessage,{customerLocation:<error className="text-danger">Please enter valid Location</error>}));
      }
      this.setState(Object.assign(this.state.formValid,{customerLocation : locationValid}));
      this.setState(Object.assign(this.state.formValid,{buttonActive : this.state.distributorName && locationValid && this.state.formValid.customerName && this.state.purchaseDate && this.state.installationDate}));
    }

    else if(fieldName == "purchaseDate"){
      var purchaseValid = false;
      var current = new Date()
      var pdate = new Date(value)
      if(value == ""){
        this.setState(Object.assign(this.state.formErrorMessage,{purchaseDate: <error className="text-danger">field required</error>}));
      }
      else if((pdate > current) || (pdate.getDate() == current.getDate() && pdate.getMonth() == current.getMonth() && pdate.getFullYear() == current.getFullYear())){
        purchaseValid = true;
        this.setState(Object.assign(this.state.formErrorMessage,{purchaseDate : ""}));}
      else{
        this.setState(Object.assign(this.state.formErrorMessage,{purchaseDate:<error className="text-danger">Purchase date should be today or greater than Todays date</error>}));
      }
      this.setState(Object.assign(this.state.formValid,{purchaseDate : purchaseValid}));
      this.setState(Object.assign(this.state.formValid,{buttonActive : this.state.distributorName && this.state.formValid.customerName && this.state.formValid.customerLocation && purchaseValid && this.state.installationDate}));
    }

    else if(fieldName == "installationDate"){
      var installationValid = false;
      var pdate = new Date(this.state.formValue.purchaseDate);
      var sdate = new Date(pdate);
      sdate.setDate(pdate.getDate() + 7);
      var idate = new Date(value);
      console.log(pdate);
      console.log(sdate);
      console.log(idate);
      if(value == ""){
        this.setState(Object.assign(this.state.formErrorMessage,{installationDate: <error className="text-danger">field required</error>}));
      }
      else if((idate <= pdate) || (idate > sdate)){
        this.setState(Object.assign(this.state.formErrorMessage,{installationDate:<error className="text-danger">Installation date should be after purchase date and within 7 days from purchase date</error>}));
      }
      else{
        installationValid = true;
        this.setState(Object.assign(this.state.formErrorMessage,{installationDate : ""}));
      }
      this.setState(Object.assign(this.state.formValid,{installationDate : installationValid}));
      this.setState(Object.assign(this.state.formValid,{buttonActive : this.state.distributorName && this.state.formValid.customerName && this.state.formValid.customerLocation && installationValid && this.state.purchaseDate}));
    }
    
    /* Perform Validations and assign error messages, Also, set the value of buttonActive after validation of the field */
  }

  render() {
    return (
      <React.Fragment>
        <div className="CreateBooking ">
          <div className="container-fluid row">
            <div className="col-md-6 offset-md-3">
              <br />
              <div className="card">
                <div className="card-header bg-custom">
                  <h4>Allocation Form</h4>
                </div>
                <div className="card-body">
                  <form className="AllocateSolar">
                    <div className="form-group">
                      <label>Distributor Name</label>
                      <select name="distributorName" id="dname" className="form-control" onChange={this.handleChange} required>
                      <option value="">--Select a Distributor--</option>
                      <option value="Suntek">Suntek</option>
                      <option value="A4Solar">A4Solar</option>
                      <option value="SupremeSolar">SupremeSolar</option>
                    </select>
                    <span name="distributorNameError"> {this.state.formErrorMessage.distributorName}</span>
                    </div>
                    <div className="form-group">
                      <label>Customer Name</label>
                      <input name="customerName" className="form-control" type="text" placeholder="Enter Customer Name" onChange={this.handleChange} required/>
                      <span name="customerNameError"> {this.state.formErrorMessage.customerName}</span>
                    </div>
                    <div className="form-group">
                      <label>Customer Location</label>
                      <input name="customerLocation" className="form-control" type="text" placeholder="Enter Customer Location" onChange={this.handleChange} required/>
                      <span name="customerLocationError"> {this.state.formErrorMessage.customerLocation}</span>
                    </div>
                    <div className="form-group">
                      <label>Purchase Date</label>
                      <input name="purchaseDate" className="form-control" type="date" onChange={this.handleChange} required/>
                      <span name="purchaseDateError"> {this.state.formErrorMessage.purchaseDate}</span>
                    </div>
                    <div className="form-group">
                      <label>Installation Date</label>
                      <input name="installationDate" className="form-control" type="date" onChange={this.handleChange} required/>
                      <span name="installationDateError"> {this.state.formErrorMessage.installationDate}</span>
                    </div><br/>
                    <button name="allocateSolar" type="button" className="btn btn-primary" disabled={!this.state.formValid.buttonActive} onClick={this.handleSubmit}>Allocate Solar</button><br/><br/>
                    <span name="successMessage" className="text-success text-bold">{this.state.successMessage}</span>
                    <span name="errorMessage" className="text-danger text-bold">{this.state.errorMessage}</span>
                  </form>
                </div>
              </div>
            </div>
          </div>
        </div>
      </React.Fragment>
    );
  }
}

export default AllocateSolar;

#App.js
import React, { Component } from "react";
import { Link } from "react-router-dom";
import { BrowserRouter as Router, Route, Switch, Redirect } from "react-router-dom";

/* Import the required modules/dependencies here */  

/* DO NOT REMOVE THIS IMPORT STATEMENT */
import AllocateSolar from "./components/AllocateSolar";
import GetDistributors from "./components/GetDistributors";
import Evaluator from './components/evaluator';
import "./App.css";
class App extends Component {
  render() {
    return (
      <Router>
      <div>
        { /* DO NOT REMOVE THIS COMPONENT TAG */}
        <Evaluator></Evaluator>

        <nav className="navbar navbar-expand-lg navbar-light  bg-custom">
          <span className="navbar-brand">IBA Retails</span>
          <ul className="navbar-nav">
            <li className="nav-item">
              <Link className="nav-link" to="/allocateSolar"> Allocate Solar </Link>
            </li>
            <li className="nav-item">
              <Link className="nav-link" to="/getDistributors"> View Distributors </Link>
            </li>
          </ul>
        </nav>
        <Switch>
          <Route exact path="/allocateSolar" component={AllocateSolar}/>
          <Route exact path="/getDistributors" component={GetDistributors}/>
          <Route path="/" render={() => (<Redirect to ="/allocateSolar"/>)}/>
        </Switch>
      </div>
      </Router>
    );
  }
}

export default App;

