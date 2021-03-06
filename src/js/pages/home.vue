<template>
  <div id="home">
    <h1 class="padding">Préparation du CV</h1>

    <div class="card">
      <div class="card-primary">
        <h1>Mes informations</h1>
        <div class="padding">
          Connectez-vous sur vos différents comptes afin de récupérer toutes vos données utiles à la génération de votre CV
        </div>
        <table class="table full-width table-centered">
          <thead>
            <tr>
              <th>Réseau</th>
              <th>Action</th>
            </tr>
          </thead>
          <tbody>
            <tr>
              <td>
                LinkedIn
              </td>
              <td>
                <button class="btn btn-primary" v-on:click="logInLinkedin" v-if="!linkedin">Connexion</button>
                <button class="btn" v-on:click="logOutLinkedin" v-else>Déconnexion</button>
              </td>
            </tr>
            <tr>
              <td>Github</td>
              <td>
                <button class="btn btn-primary" v-on:click="setGithubUsername" v-if="!github">Connexion</button>
                <button class="btn" v-on:click="resetGithub" v-else>Déconnexion</button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
      <div class="card-actions">
        <button class="btn float-right" v-on:click="logout" v-if="linkedin || github">Déconnexion</button>
      </div>
    </div>

    <div class="card">
      <div class="card-primary">
        <h1>Choix des informations</h1>
      </div>
      <div class="card-secondary">
        <div style="margin-bottom: 10px">
          <label for="select">Choix de la langue</label>
          <select id="select">
            <option value="fr" selected>Français</option>
            <option value="en">English</option>
          </select>
        </div>
        <label>Choix de fusion des données</label>
        <table class="table full-width table-centered">
          <thead>
            <tr>
              <th>Réseau</th>
              <th>Avatar</th>
              <th>Mail</th>
            </tr>
          </thead>
          <tbody>
            <tr>
              <td>
                LinkedIn
              </td>
              <td>
                <input type="radio" name="avatar" v-model="avatar" value="linkedin">
              </td>
              <td>
                <input type="radio" name="mail" v-model="mail" value="linkedin">
              </td>
            </tr>
            <tr>
              <td>Github</td>
              <td>
                <input type="radio" name="avatar" v-model="avatar" value="github">
              </td>
              <td>
                <input type="radio" name="mail" v-model="mail" value="github">
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <div class="padding text-center">
      <button class="btn btn-large" v-on:click="exportToXML">Exporter en XML</button>
      <button class="btn btn-large btn-primary" v-on:click="nextStep">Générer mon CV</button>
    </div>

    <div style="visibility: hidden">
      <script id="linkedinBtn" type="in/Login"></script>
    </div>
  </div>
</template>


<script>
import XMLParser from '../tools/XMLParser';
import Github from '../tools/Github';


export default {
  
  data() {
    return {
      avatar: 'github',
      mail: 'linkedin',
      linkedin: undefined,
      github: undefined
    }
  },

  created() {
    let githubUsername = localStorage.getItem('github_username'),
        elt = this;

    Linkedin.getProfile(profile => {
      elt.linkedin = profile;
    })

    if (githubUsername) {
      this.searchGithubInformations(githubUsername)
    }
  },

  methods: {
    searchGithubInformations(username) {
      let elt = this;

      Github.getAllUserDatas(username, (err, user) => {
        if (!err) {
          localStorage.setItem('github_username', username)
          elt.github = user;
        } else {
          alert('Erreur lors de la récupération des données Github')
          console.log(err)
        }
      })
    },

    setGithubUsername() {
      const username = prompt('Quel est votre pseudo Github ?')

      this.searchGithubInformations(username)
    },

    resetGithub() {
      localStorage.removeItem('github_username')
      this.github = undefined;
    },

    logInLinkedin() {
      document
        .getElementById('linkedinBtn')
        .previousSibling
        .getElementsByTagName('a')[0]
        .click()
    },

    logOutLinkedin() {
      let elt = this;
      Linkedin.logOut(() => {
        elt.linkedin = undefined;
      })
    },

    logout() {
      this.resetGithub()
      this.logOutLinkedin()
    },

    checkData() {
      let errorDetails = '';
      if(!this.github || !this.linkedin){
        if(!this.github) {errorDetails += ' github ';}
        if(!this.linkedin) {errorDetails += ' linkedin ';}
        errorDetails = `Impossible de récupérer les données de : ${errorDetails}`;
        return errorDetails;
      }
      else if(!this.avatar || !this.mail){
        if(!this.avatar) {errorDetails += ' avatar ';}
        if(!this.mail) {errorDetails += ' mail ';}
        errorDetails = `Choisir la source de données pour : ${errorDetails}`;
        return errorDetails;
      }
      else if(this.github.email === null || this.linkedin.emailAddress === null) {

        if(this.mail === 'github' && this.github.email === null) {
          errorDetails += ' github ';
          errorDetails = `La source de données : ${errorDetails} ne renvoie pas de mail merci de choisir une source différente`;
          return errorDetails;
        }
        if(this.mail === 'linkedin' && this.linkedin.email === null) {
          errorDetails += ' linkedin ';
          errorDetails = `La source de données : ${errorDetails} ne renvoie pas de mail merci de choisir une source différente`;
          return errorDetails;
        }
        return true;
      }
      else {return true;}

    },

    saveUserInformation() {
      let msg;

      if ((msg = this.checkData()) === true) {
        let user = {
            firstName: this.linkedin.firstName,
            lastName: this.linkedin.lastName,
            name: this.github.name,
            picture: this.avatar === 'github' ? this.github.image : this.linkedin.pictureUrl,
            experiences: this.linkedin.positions.values,
            projects: this.github.repos,
            mail: this.mail === 'github' ? this.github.email : this.linkedin.emailAddress 
          };

        sessionStorage.setItem('user', JSON.stringify(user))
      } else {
        alert(msg)
      }
    },

    exportToXML() {
      try {
        this.saveUserInformation()

        let xml = XMLParser.fromJson(sessionStorage.getItem('user'), 'cv', 'schema.xsd');

        XMLParser.generateDownloadLink('cv.xml', xml)
      } catch (err) {
        console.log(err)
        alert('Erreur lors de la génération du XML')
      }
    },

    nextStep() {
      this.saveUserInformation()
      this.$router.push({ name: 'cvbase' })
    }
  }

}
</script>
