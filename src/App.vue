<template>
  <div class="quiz-app" :class="{ 'dark-mode': isDarkMode }">
    <div class="theme-toggle">
      <button @click="toggleDarkMode" class="theme-btn">
        {{ isDarkMode ? '☀️' : '🌙' }}
      </button>
    </div>
    <h1>{{ title }}</h1>
    
    <!-- Écran de démarrage -->
    <div v-if="currentScreen === 'start'" class="start-screen">
      <h2>Bienvenue au quiz!</h2>
      <p>Testez vos connaissances avec ce quiz interactif.</p>
      
      <div class="settings">
        <div class="setting-group">
          <label>Difficulté:</label>
          <div class="difficulty-selector">
            <button 
              v-for="level in difficultyLevels" 
              :key="level"
              @click="selectedDifficulty = level"
              :class="['difficulty-btn', { active: selectedDifficulty === level }]"
            >
              {{ level }}
            </button>
          </div>
        </div>
        
        <div class="setting-group">
          <label>Catégorie:</label>
          <select v-model="selectedCategory" class="category-select">
            <option v-for="category in categories" :key="category.id" :value="category.id">
              {{ category.name }}
            </option>
          </select>
        </div>
        
        <div class="setting-group">
          <label>Temps par question (secondes):</label>
          <select v-model="timePerQuestion" class="time-select">
            <option :value="10">10</option>
            <option :value="20">20</option>
            <option :value="30">30</option>
            <option :value="45">45</option>
            <option :value="60">60</option>
          </select>
        </div>
      </div>
      
      <button @click="startQuiz" class="btn">Commencer le quiz</button>
      
      <div v-if="highScores.length > 0" class="high-scores">
        <h3>Meilleurs scores</h3>
        <ul>
          <li v-for="(score, index) in highScores.slice(0, 5)" :key="index">
            {{ score.category }} ({{ score.difficulty }}) - {{ score.score }}% - {{ new Date(score.date).toLocaleDateString() }}
          </li>
        </ul>
      </div>
    </div>
    
    <!-- Écran de questions -->
    <div v-else-if="currentScreen === 'quiz'" class="quiz-screen">
      <transition name="fade" mode="out-in">
        <div :key="currentQuestionIndex" class="quiz-content">
          <div class="progress">
            <div class="progress-text">
              Question {{ currentQuestionIndex + 1 }} sur {{ filteredQuestions.length }}
              <span class="timer">
                Temps restant: {{ remainingTime }}s
              </span>
            </div>
            <div class="progress-bar">
              <div class="progress-fill" :style="{ width: progressPercentage + '%' }"></div>
            </div>
            <div class="timer-bar">
              <div class="timer-fill" :style="{ width: (remainingTime / timePerQuestion) * 100 + '%' }"></div>
            </div>
          </div>
          
          <div class="question-container">
            <h2>{{ currentQuestion.question }}</h2>
            
            <div v-if="currentQuestion.image" class="question-image">
              <img :src="currentQuestion.image" :alt="'Image pour ' + currentQuestion.question">
            </div>
            
            <div class="options">
              <div 
                v-for="(option, index) in currentQuestion.options" 
                :key="index" 
                @click="selectAnswer(index)"
                :class="['option', { 
                  'selected': selectedOptionIndex === index,
                  'correct': showFeedback && index === currentQuestion.correctIndex,
                  'incorrect': showFeedback && selectedOptionIndex === index && index !== currentQuestion.correctIndex
                }]"
              >
                {{ option }}
              </div>
            </div>
          </div>
          
          <div v-if="showFeedback" class="feedback">
            <p v-if="isCorrect" class="correct-feedback">Correct!</p>
            <p v-else class="incorrect-feedback">
              Incorrect. La bonne réponse est: {{ currentQuestion.options[currentQuestion.correctIndex] }}
            </p>
            <p v-if="currentQuestion.explanation" class="explanation">
              {{ currentQuestion.explanation }}
            </p>
          </div>
          
          <button @click="nextQuestion" class="btn" :disabled="selectedOptionIndex === null && !timeExpired">
            {{ currentQuestionIndex === filteredQuestions.length - 1 ? 'Voir les résultats' : 'Question suivante' }}
          </button>
        </div>
      </transition>
    </div>
    
    <!-- Écran de résultats -->
    <div v-else-if="currentScreen === 'results'" class="results-screen">
      <h2>Quiz terminé!</h2>
      <div class="score">
        <p>Votre score: <strong>{{ score }} sur {{ filteredQuestions.length }}</strong></p>
        <p>Pourcentage: <strong>{{ scorePercentage }}%</strong></p>
        <p>Difficulté: <strong>{{ selectedDifficulty }}</strong></p>
        <p>Catégorie: <strong>{{ getCategoryName(selectedCategory) }}</strong></p>
        <p>Temps moyen par question: <strong>{{ averageTimePerQuestion }}s</strong></p>
      </div>
      
      <div class="results-summary">
        <h3>Résumé:</h3>
        <ul>
          <li v-for="(question, index) in filteredQuestions" :key="index">
            <span :class="userAnswers[index] === question.correctIndex ? 'correct' : 'incorrect'">
              Question {{ index + 1 }}: {{ userAnswers[index] === question.correctIndex ? 'Correct' : 'Incorrect' }}
              <span class="question-time">({{ questionTimes[index] }}s)</span>
            </span>
          </li>
        </ul>
      </div>
      
      <div class="action-buttons">
        <button @click="restartQuiz" class="btn">Recommencer le quiz</button>
        <button @click="goToStart" class="btn btn-secondary">Changer de paramètres</button>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'QuizApp',
  data() {
    return {
      // Titre de l'application
      title: 'Quiz App',
      
      // État de l'application: 'start', 'quiz', 'results'
      currentScreen: 'start',
      
      // Index de la question actuelle
      currentQuestionIndex: 0,
      
      // Option sélectionnée par l'utilisateur
      selectedOptionIndex: null,
      
      // Afficher le feedback après la réponse
      showFeedback: false,
      
      // Tableau pour stocker les réponses de l'utilisateur
      userAnswers: [],
      
      // Nouvelles fonctionnalités
      isDarkMode: false,
      selectedDifficulty: 'Facile',
      difficultyLevels: ['Facile', 'Moyen', 'Difficile'],
      selectedCategory: 'general',
      timePerQuestion: 30,
      remainingTime: 30,
      timerInterval: null,
      timeExpired: false,
      questionTimes: [],
      questionStartTime: 0,
      highScores: [],
      
      // Catégories
      categories: [
        { id: 'general', name: 'Culture Générale' },
        { id: 'vue', name: 'Vue.js' },
        { id: 'histoire', name: 'Histoire' },
        { id: 'sciences', name: 'Sciences' }
      ],
      
      // Questions du quiz étendues avec difficulté, catégorie et explications
      allQuestions: [
        // Catégorie: Culture Générale
        {
          question: "Quelle est la capitale de la France?",
          options: ["Londres", "Berlin", "Paris", "Madrid"],
          correctIndex: 2,
          difficulty: 'Facile',
          category: 'general',
          explanation: "Paris est la capitale de la France depuis plus de 1000 ans."
        },
        {
          question: "Quel est le plus grand océan du monde?",
          options: ["Atlantique", "Indien", "Arctique", "Pacifique"],
          correctIndex: 3,
          difficulty: 'Facile',
          category: 'general',
          explanation: "L'océan Pacifique couvre environ 46% de la surface aquatique de la Terre."
        },
        {
          question: "Quelle est la plus grande planète du système solaire?",
          options: ["Mars", "Jupiter", "Saturne", "Vénus"],
          correctIndex: 1,
          difficulty: 'Facile',
          category: 'general',
          explanation: "Jupiter est la plus grande planète, avec une masse 318 fois supérieure à celle de la Terre."
        },
        {
          question: "Quel élément chimique a pour symbole 'Au'?",
          options: ["Argent", "Or", "Aluminium", "Argon"],
          correctIndex: 1,
          difficulty: 'Moyen',
          category: 'general',
          explanation: "Le symbole 'Au' vient du mot latin 'Aurum' qui signifie or."
        },
        {
          question: "Quelle est la capitale de l'Australie?",
          options: ["Sydney", "Melbourne", "Canberra", "Perth"],
          correctIndex: 2,
          difficulty: 'Moyen',
          category: 'general',
          explanation: "Bien que Sydney soit la plus grande ville, Canberra est la capitale de l'Australie depuis 1908."
        },
        {
          question: "Quel est le théorème mathématique qui établit que dans un triangle rectangle, le carré de l'hypoténuse est égal à la somme des carrés des deux autres côtés?",
          options: ["Théorème de Fermat", "Théorème de Pythagore", "Théorème de Thalès", "Théorème d'Euclide"],
          correctIndex: 1,
          difficulty: 'Difficile',
          category: 'general',
          explanation: "Le théorème de Pythagore est fondamental en géométrie: a² + b² = c² (où c est l'hypoténuse)."
        },
        
        // Catégorie: Vue.js
        {
          question: "Quel est le framework JavaScript utilisé dans cette application?",
          options: ["React", "Angular", "Vue.js", "Svelte"],
          correctIndex: 2,
          difficulty: 'Facile',
          category: 'vue',
          explanation: "Vue.js est un framework JavaScript progressif créé par Evan You."
        },
        {
          question: "Quelle méthode est utilisée pour réagir à un clic sur un bouton dans Vue.js?",
          options: ["v-click", "@click", "v-on:click", "B et C sont corrects"],
          correctIndex: 3,
          difficulty: 'Facile',
          category: 'vue',
          explanation: "@click est un raccourci pour v-on:click, les deux sont corrects."
        },
        {
          question: "Quelle directive Vue.js est utilisée pour le rendu conditionnel?",
          options: ["v-if", "v-show", "v-for", "v-render"],
          correctIndex: 0,
          difficulty: 'Moyen',
          category: 'vue',
          image: "/api/placeholder/300/200",
          explanation: "v-if supprime/ajoute complètement les éléments du DOM, tandis que v-show utilise display:none."
        },
        {
          question: "Quel est le système de gestion d'état recommandé pour les applications Vue.js complexes?",
          options: ["Redux", "Context API", "Vuex", "MobX"],
          correctIndex: 2,
          difficulty: 'Moyen',
          category: 'vue',
          explanation: "Vuex est le gestionnaire d'état officiel pour Vue.js, bien que Pinia soit maintenant recommandé pour Vue 3."
        },
        {
          question: "Comment créer une computed property dans Vue.js?",
          options: [
            "Utiliser la méthode computed()",
            "Définir une fonction dans l'objet computed",
            "Ajouter un attribut @computed à une propriété",
            "Utiliser la directive v-computed"
          ],
          correctIndex: 1,
          difficulty: 'Difficile',
          category: 'vue',
          explanation: "Les computed properties sont définies comme des fonctions dans l'objet computed, mais sont utilisées comme des propriétés."
        },
        
        // Catégorie: Histoire
        {
          question: "En quelle année a eu lieu la révolution française?",
          options: ["1789", "1804", "1776", "1815"],
          correctIndex: 0,
          difficulty: 'Facile',
          category: 'histoire',
          explanation: "La Révolution française a commencé en 1789 avec la prise de la Bastille le 14 juillet."
        },
        {
          question: "Qui était le premier président des États-Unis?",
          options: ["John Adams", "Thomas Jefferson", "Benjamin Franklin", "George Washington"],
          correctIndex: 3,
          difficulty: 'Facile',
          category: 'histoire',
          image: "/api/placeholder/300/200",
          explanation: "George Washington a servi comme premier président de 1789 à 1797."
        },
        {
          question: "Quelle civilisation a construit les pyramides de Gizeh?",
          options: ["Les Grecs", "Les Romains", "Les Égyptiens", "Les Mayas"],
          correctIndex: 2,
          difficulty: 'Moyen',
          category: 'histoire',
          explanation: "Les pyramides de Gizeh ont été construites par les Égyptiens pendant l'Ancien Empire, il y a environ 4500 ans."
        },
        {
          question: "Quel traité a mis fin à la Première Guerre mondiale?",
          options: ["Traité de Versailles", "Traité de Paris", "Traité de Westphalie", "Traité de Rome"],
          correctIndex: 0,
          difficulty: 'Moyen',
          category: 'histoire',
          explanation: "Le Traité de Versailles, signé en 1919, a officiellement mis fin à la guerre entre l'Allemagne et les Alliés."
        },
        {
          question: "Quelle bataille navale décisive a eu lieu en 1805 pendant les guerres napoléoniennes?",
          options: ["Bataille de Waterloo", "Bataille de Trafalgar", "Bataille d'Austerlitz", "Bataille de Borodino"],
          correctIndex: 1,
          difficulty: 'Difficile',
          category: 'histoire',
          explanation: "La bataille de Trafalgar, où l'amiral Nelson a vaincu les flottes française et espagnole, établissant la domination navale britannique."
        },
        
        // Catégorie: Sciences
        {
          question: "Quelle est la formule chimique de l'eau?",
          options: ["CO2", "H2O", "O2", "NaCl"],
          correctIndex: 1,
          difficulty: 'Facile',
          category: 'sciences',
          explanation: "H2O signifie que chaque molécule d'eau est composée de deux atomes d'hydrogène et un d'oxygène."
        },
        {
          question: "Quelle est la planète la plus proche du Soleil?",
          options: ["Vénus", "Mars", "Mercure", "Jupiter"],
          correctIndex: 2,
          difficulty: 'Facile',
          category: 'sciences',
          explanation: "Mercure est la première planète du système solaire, à une distance moyenne de 58 millions de km du Soleil."
        },
        {
          question: "Quelle est l'unité de mesure de la force dans le Système International?",
          options: ["Watt", "Joule", "Pascal", "Newton"],
          correctIndex: 3,
          difficulty: 'Moyen',
          category: 'sciences',
          explanation: "Le Newton (N) est l'unité SI de force, défini comme la force nécessaire pour accélérer une masse de 1 kg à 1 m/s²."
        },
        {
          question: "Quel est le processus par lequel les plantes convertissent la lumière en énergie?",
          options: ["Photosynthèse", "Respiration", "Fermentation", "Combustion"],
          correctIndex: 0,
          difficulty: 'Moyen',
          category: 'sciences',
          image: "/api/placeholder/300/200",
          explanation: "La photosynthèse utilise la lumière du soleil, l'eau et le CO2 pour produire de l'oxygène et du glucose."
        },
        {
          question: "Quelle est la constante de Planck, fondamentale en mécanique quantique?",
          options: ["6.022 × 10²³", "3.14159", "6.626 × 10⁻³⁴", "2.998 × 10⁸"],
          correctIndex: 2,
          difficulty: 'Difficile',
          category: 'sciences',
          explanation: "La constante de Planck (h) a une valeur de 6.626 × 10⁻³⁴ joule-seconde et est essentielle en physique quantique."
        }
      ]
    };
  },
  
  computed: {
    // Filtrer les questions selon la difficulté et la catégorie
    filteredQuestions() {
      return this.allQuestions.filter(q => 
        q.difficulty === this.selectedDifficulty && 
        q.category === this.selectedCategory
      );
    },
    
    // Obtenir la question actuelle
    currentQuestion() {
      return this.filteredQuestions[this.currentQuestionIndex];
    },
    
    // Calculer la progression du quiz en pourcentage
    progressPercentage() {
      return (this.currentQuestionIndex / this.filteredQuestions.length) * 100;
    },
    
    // Vérifier si la réponse sélectionnée est correcte
    isCorrect() {
      return this.selectedOptionIndex === this.currentQuestion.correctIndex;
    },
    
    // Calculer le score final
    score() {
      return this.userAnswers.filter((answer, index) => 
        answer === this.filteredQuestions[index].correctIndex
      ).length;
    },
    
    // Calculer le pourcentage du score
    scorePercentage() {
      return Math.round((this.score / this.filteredQuestions.length) * 100);
    },
    
    // Calculer le temps moyen par question
    averageTimePerQuestion() {
      if (this.questionTimes.length === 0) return 0;
      const sum = this.questionTimes.reduce((acc, time) => acc + time, 0);
      return Math.round(sum / this.questionTimes.length);
    }
  },
  
  methods: {
    // Basculer entre mode clair et sombre
    toggleDarkMode() {
      this.isDarkMode = !this.isDarkMode;
      localStorage.setItem('darkMode', this.isDarkMode);
    },
    
    // Obtenir le nom de la catégorie à partir de son ID
    getCategoryName(categoryId) {
      const category = this.categories.find(cat => cat.id === categoryId);
      return category ? category.name : '';
    },
    
    // Démarrer le quiz
    startQuiz() {
      if (this.filteredQuestions.length === 0) {
        alert("Aucune question disponible pour cette combinaison de difficulté et catégorie. Veuillez choisir une autre option.");
        return;
      }
      
      this.currentScreen = 'quiz';
      this.resetQuiz();
      this.startTimer();
      this.questionStartTime = Date.now();
    },
    
    // Démarrer le minuteur
    startTimer() {
      this.remainingTime = this.timePerQuestion;
      clearInterval(this.timerInterval);
      this.timeExpired = false;
      
      this.timerInterval = setInterval(() => {
        this.remainingTime--;
        if (this.remainingTime <= 0) {
          clearInterval(this.timerInterval);
          this.timeExpired = true;
          
          // Si aucune réponse n'a été sélectionnée, compter comme incorrect
          if (this.selectedOptionIndex === null) {
            this.userAnswers[this.currentQuestionIndex] = -1; // -1 pour indiquer "pas de réponse"
            this.showFeedback = true;
            
            // Enregistrer le temps maximum
            this.recordQuestionTime();
          }
        }
      }, 1000);
    },
    
    // Enregistrer le temps pris pour répondre à la question
    recordQuestionTime() {
      const timeTaken = Math.min(this.timePerQuestion - this.remainingTime, this.timePerQuestion);
      this.questionTimes[this.currentQuestionIndex] = timeTaken;
    },
    
    // Sélectionner une réponse
    selectAnswer(index) {
      // Empêcher la modification si le feedback est déjà affiché ou si le temps est écoulé
      if (this.showFeedback || this.timeExpired) return;
      
      this.selectedOptionIndex = index;
      this.showFeedback = true;
      
      // Enregistrer le temps pris pour répondre
      this.recordQuestionTime();
      
      // Arrêter le minuteur
      clearInterval(this.timerInterval);
      
      // Sauvegarder la réponse
      this.userAnswers[this.currentQuestionIndex] = index;
    },
    
    // Passer à la question suivante ou afficher les résultats
    nextQuestion() {
      this.showFeedback = false;
      this.selectedOptionIndex = null;
      
      if (this.currentQuestionIndex < this.filteredQuestions.length - 1) {
        this.currentQuestionIndex++;
        this.startTimer();
        this.questionStartTime = Date.now();
      } else {
        this.currentScreen = 'results';
        this.saveHighScore();
      }
    },
    
    // Sauvegarder le score
    saveHighScore() {
      const newScore = {
        score: this.scorePercentage,
        difficulty: this.selectedDifficulty,
        category: this.getCategoryName(this.selectedCategory),
        date: Date.now()
      };
      
      let highScores = JSON.parse(localStorage.getItem('highScores') || '[]');
      highScores.push(newScore);
      
      // Trier par score et ne garder que les 10 meilleurs
      highScores.sort((a, b) => b.score - a.score);
      highScores = highScores.slice(0, 10);
      
      localStorage.setItem('highScores', JSON.stringify(highScores));
      this.highScores = highScores;
    },
    
    // Réinitialiser et recommencer le quiz
    restartQuiz() {
      this.resetQuiz();
      this.currentScreen = 'quiz';
      this.startTimer();
      this.questionStartTime = Date.now();
    },
    
    // Retourner à l'écran de démarrage
    goToStart() {
      this.currentScreen = 'start';
    },
    
    // Réinitialiser les variables du quiz
    resetQuiz() {
      this.currentQuestionIndex = 0;
      this.selectedOptionIndex = null;
      this.showFeedback = false;
      this.userAnswers = Array(this.filteredQuestions.length).fill(null);
      this.questionTimes = Array(this.filteredQuestions.length).fill(0);
      this.timeExpired = false;
    },
    
    // Charger les préférences enregistrées
    loadSavedPreferences() {
      const savedDarkMode = localStorage.getItem('darkMode');
      if (savedDarkMode !== null) {
        this.isDarkMode = savedDarkMode === 'true';
      }
      
      const savedHighScores = localStorage.getItem('highScores');
      if (savedHighScores) {
        this.highScores = JSON.parse(savedHighScores);
      }
    }
  },
  
  // Charger les préférences au montage du composant
  mounted() {
    this.loadSavedPreferences();
  },
  
  // Nettoyer les intervalles lorsque le composant est détruit
  beforeUnmount() {
    clearInterval(this.timerInterval);
  }
};
</script>

<style scoped>
.quiz-app {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Arial', sans-serif;
  color: #2c3e50;
  transition: all 0.3s ease;
}

.quiz-app.dark-mode {
  background-color: #222;
  color: #f0f0f0;
}

.dark-mode .question-container,
.dark-mode .start-screen,
.dark-mode .results-summary {
  background-color: #333;
  color: #f0f0f0;
}

.dark-mode .option {
  background-color: #444;
  color: #f0f0f0;
  border-color: #666;
}

.dark-mode .option:hover {
  background-color: #555;
}

.dark-mode .option.selected {
  border-color: #42b983;
  background-color: rgba(66, 185, 131, 0.2);
}

.theme-toggle {
  text-align: right;
  margin-bottom: 10px;
}

.theme-btn {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  transition: transform 0.3s ease;
}

.theme-btn:hover {
  transform: scale(1.2);
}

h1 {
  text-align: center;
  color: #42b983;
  margin-bottom: 30px;
}

.dark-mode h1 {
  color: #5cd9a0;
}

.btn {
  background-color: #42b983;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  margin-top: 20px;
  transition: all 0.3s;
}

.btn:hover {
  background-color: #3aa876;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0,0,0,0.1);
}

.btn:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

.btn-secondary {
  background-color: #7f8c8d;
  margin-left: 10px;
}

.btn-secondary:hover {
  background-color: #6c7a7a;
}

/* Styles pour les paramètres */
.settings {
  background-color: #f9f9f9;
  padding: 15px;
  border-radius: 8px;
  margin: 20px 0;
}

.dark-mode .settings {
  background-color: #333;
}

.setting-group {
  margin-bottom: 15px;
}

.setting-group label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
}

.difficulty-selector {
  display: flex;
  gap: 10px;
}

.difficulty-btn {
  padding: 8px 15px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background-color: #f0f0f0;
  cursor: pointer;
  transition: all 0.2s;
}

.dark-mode .difficulty-btn {
  background-color: #444;
  border-color: #666;
  color: #f0f0f0;
}

.difficulty-btn:hover {
  background-color: #e0e0e0;
}

.dark-mode .difficulty-btn:hover {
  background-color: #555;
}

.difficulty-btn.active {
  background-color: #42b983;
  color: white;
  border-color: #42b983;
}

.dark-mode .difficulty-btn.active {
  background-color: #5cd9a0;
}

.category-select,
.time-select {
  width: 100%;
  padding: 8px;
  border-radius: 4px;
  border: 1px solid #ddd;
}

.dark-mode .category-select,
.dark-mode .time-select {
  background-color: #444;
  color: #f0f0f0;
  border-color: #666;
}

/* Styles pour la barre de progression et le minuteur */
.progress {
  margin-bottom: 20px;
}

.progress-text {
  display: flex;
  justify-content: space-between;
  margin-bottom: 5px;
  font-size: 14px;
  color: #666;
}

.dark-mode .progress-text {
  color: #aaa;
}

.timer {
  font-weight: bold;
}

.progress-bar,
.timer-bar {
  background-color: #f0f0f0;
  height: 10px;
  border-radius: 5px;
  overflow: hidden;
  margin-bottom: 5px;
}

.dark-mode .progress-bar,
.dark-mode .timer-bar {
  background-color: #444;
}

.progress-fill {
  background-color: #42b983;
  height: 100%;
  transition: width 0.3s ease;
}

.timer-fill {
  background-color: #e74c3c;
  height: 100%;
  transition: width 1s linear;
}

.dark-mode .timer-fill {
  background-color: #ff6b6b;
}

/* Styles pour les questions */
.question-container {
  background-color: #f9f9f9;
  padding: 20px;
  border-radius: 8px;
  margin-bottom: 20px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.05);
}

.question-image {
  margin: 15px 0;
  text-align: center;
}

.question-image img {
  max-width: 100%;
  border-radius: 4px;
  box-shadow: 0 2px 5px rgba(0,0,0,0.1);
}

.options {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-top: 15px;
}

.option {
  padding: 12px 15px;
  background-color: #fff;
  border: 1px solid #ddd;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.2s;
}

.option:hover {
  background-color: #f0f0f0;
  transform: translateX(5px);
}

.option.selected {
  border-color: #42b983;
  background-color: rgba(66, 185, 131, 0.1);
}

.option.correct {
  background-color: rgba(76, 175, 80, 0.2);
  border-color: #4CAF50;
}

.option.incorrect {
  background-color: rgba(244, 67, 54, 0.2);
  border-color: #F44336;
}

/* Styles pour le feedback */
.feedback {
  margin-top: 15px;
  padding: 15px;
  border-radius: 4px;
  background-color: #f9f9f9;
  box-shadow: 0 2px 5px rgba(0,0,0,0.05);
}

.dark-mode .feedback {
  background-color: #333;
}

.correct-feedback {
  color: #4CAF50;
  font-weight: bold;
}

.incorrect-feedback {
  color: #F44336;
}

.explanation {
  margin-top: 10px;
  font-style: italic;
  color: #666;
}

.dark-mode .explanation {
  color: #bbb;
}

/* Styles pour les résultats */
.results-screen {
  text-align: center;
}

.score {
  font-size: 18px;
  margin: 20px 0;
  line-height: 1.6;
}

.results-summary {
  background-color: #f9f9f9;
  padding: 20px;
  border-radius: 8px;
  margin: 20px auto;
  max-width: 500px;
  text-align: left;
  box-shadow: 0 2px 10px rgba(0,0,0,0.05);
}

.results-summary ul {
  list-style-type: none;
  padding: 0;
}

.results-summary li {
  padding: 8px 0;
  border-bottom: 1px solid #eee;
}

.dark-mode .results-summary li {
  border-bottom-color: #444;
}

.results-summary .correct {
  color: #4CAF50;
}

.results-summary .incorrect {
  color: #F44336;
}

.question-time {
  margin-left: 10px;
  font-size: 0.9em;
  color: #777;
}

.dark-mode .question-time {
  color: #aaa;
}

/* Styles pour l'écran de démarrage */
.start-screen {
  text-align: center;
  padding: 40px 20px;
  background-color: #f9f9f9;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.05);
}

/* Styles pour les meilleurs scores */
.high-scores {
  margin-top: 30px;
  padding: 15px;
  background-color: #f5f5f5;
  border-radius: 8px;
}

.dark-mode .high-scores {
  background-color: #333;
}

.high-scores h3 {
  margin-top: 0;
  color: #42b983;
}

.high-scores ul {
  padding: 0;
  list-style-type: none;
}

.high-scores li {
  padding: 5px 0;
  border-bottom: 1px solid #eee;
}

.dark-mode .high-scores li {
  border-bottom-color: #444;
}

/* Animation de transition */
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.5s, transform 0.5s;
}

.fade-enter-from, .fade-leave-to {
  opacity: 0;
  transform: translateX(20px);
}

/* Styles pour les boutons d'action */
.action-buttons {
  display: flex;
  justify-content: center;
  gap: 10px;
  margin-top: 20px;
}

/* Responsive design */
@media screen and (max-width: 600px) {
  .quiz-app {
    padding: 10px;
  }
  
  .difficulty-selector {
    flex-direction: column;
  }
  
  .action-buttons {
    flex-direction: column;
  }
  
  .btn-secondary {
    margin-left: 0;
  }
}
</style>