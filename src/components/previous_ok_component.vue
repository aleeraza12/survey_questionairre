<template>
    <div class="therapeutic-container" :class="{ 'stoplight-survey': isStoplightSurvey }">
      <!-- Auto-initialization trigger -->
      <div v-if="!initialized" style="display: none;">{{ autoInit }}</div>
      
      <!-- Gentle Background Elements -->
      <div class="floating-elements">
        <div class="floating-circle circle-1"></div>
        <div class="floating-circle circle-2"></div>
        <div class="floating-circle circle-3"></div>
      </div>
  
      <!-- Loading State -->
      <div v-if="loading" class="loading-container">
        <div class="loading-dots">
          <div class="dot"></div>
          <div class="dot"></div>
          <div class="dot"></div>
        </div>
        <h2 class="loading-title">Preparing Your Assessment</h2>
        <p class="loading-text">Loading personalized questions for you...</p>
      </div>
  
      <!-- Error State -->
      <div v-if="!loading && questions.length === 0" class="error-container">
        <div class="error-icon">⚠️</div>
        <h2 class="error-title">Unable to Load Assessment</h2>
        <p class="error-message">We're having trouble loading the assessment questions. Please try refreshing the page or contact support if the problem persists.</p>
        <button @click="fetchSurveyData" class="retry-button">
          <span class="button-icon">🔄</span>
          <span>Try Again</span>
        </button>
      </div>
  
      <!-- Main Content Wrapper for 2-column layout -->
      <div v-if="!loading && !showResults && questions.length > 0" class="main-content-wrapper">
        <!-- Left Column with Header -->
        <div class="left-column">
          <!-- Survey Configuration Dropdowns - Above left column on laptop screens -->
          <div class="survey-config-container">
            <div class="config-dropdowns">
              <select v-model="selectedShift" class="config-dropdown" :class="{ 'required-field': !selectedShift }">
                <option value="">Select Shift *</option>
                <option value="morning">Morning Shift</option>
                <option value="night">Night Shift</option>
              </select>
              
              <select v-model="selectedLocation" @change="onDeviceSelection" class="config-dropdown" :class="{ 'required-field': !selectedLocation }" :disabled="loadingDevices">
                <option value="">{{ loadingDevices ? 'Loading devices...' : 'Select Device *' }}</option>
                <option v-for="device in stationDevices" :key="device.stationID" :value="device.customLabel1">
                  {{ device.customLabel1 }}
                </option>
              </select>
            </div>
          </div>
          <!-- Supportive Header -->
          <div class="gentle-header">
          <div class="breathing-icon">
            <div class="breath-circle"></div>
          </div>
          
          <h1 class="caring-title">{{ surveyData.surveyName || 'Patient Assessment' }}</h1>
          <p class="supportive-message" style="display: none;">How are you today?</p>
          <p class="encouragement">Take a moment to check in with yourself. Your feelings matter. 💙</p>
          
          <!-- Progress Section - Inside Header for 2-column layout -->
          <div v-if="!loading && questions.length > 0" class="progress-section">
          <!-- Circular Progress for Laptop -->
          <div class="circular-progress-container">
            <div class="circular-progress-wrapper">
              <svg class="circular-progress" width="120" height="120" viewBox="0 0 120 120">
                <defs>
                  <linearGradient id="progressGradient" x1="0%" y1="0%" x2="100%" y2="0%">
                    <stop offset="0%" style="stop-color:#667eea;stop-opacity:1" />
                    <stop offset="25%" style="stop-color:#764ba2;stop-opacity:1" />
                    <stop offset="50%" style="stop-color:#f093fb;stop-opacity:1" />
                    <stop offset="75%" style="stop-color:#f5576c;stop-opacity:1" />
                    <stop offset="100%" style="stop-color:#4facfe;stop-opacity:1" />
                  </linearGradient>
                </defs>
                <circle 
                  class="progress-bg"
                  cx="60" 
                  cy="60" 
                  r="50" 
                  stroke-width="8"
                  fill="none"
                />
                <circle 
                  class="progress-circle"
                  cx="60" 
                  cy="60" 
                  r="50" 
                  stroke-width="8"
                  fill="none"
                  :stroke-dasharray="314.16"
                  :stroke-dashoffset="314.16 - (314.16 * progressPercentage / 100)"
                  stroke="url(#progressGradient)"
                />
              </svg>
              <div class="circular-progress-content">
                <div class="progress-percentage">{{ Math.round(progressPercentage) }}%</div>
                <div class="progress-step">{{ currentStep + 1 }} / {{ totalSteps }}</div>
              </div>
            </div>
          </div>
          
          <!-- Linear Progress for Mobile/Tablet -->
          <div class="linear-progress-container">
            <div class="progress-bar-header">
              <span class="progress-text">Progress: {{ Math.round(progressPercentage) }}%</span>
              <span class="step-counter">{{ currentStep + 1 }} / {{ totalSteps }}</span>
            </div>
            <div class="progress-bar-track">
              <div 
                class="progress-bar-fill" 
                :style="{ width: progressPercentage + '%' }"
              >
                <div class="progress-bar-glow"></div>
              </div>
            </div>
          </div>
          </div>
          </div>
        </div>
  
        <!-- Right Column with Question Card -->
        <div class="right-column">
          <!-- Main Content Area -->
          <div class="question-sanctuary">
            <!-- Survey Configuration Dropdowns - For smaller screens -->
            <div class="survey-config-container-mobile">
              <div class="config-dropdowns">
              <select v-model="selectedShift" class="config-dropdown" :class="{ 'required-field': !selectedShift }">
                <option value="">Select Shift *</option>
                <option value="morning">Morning Shift</option>
                <option value="night">Night Shift</option>
              </select>
              
              <select v-model="selectedLocation" @change="onDeviceSelection" class="config-dropdown" :class="{ 'required-field': !selectedLocation }" :disabled="loadingDevices">
                <option value="">{{ loadingDevices ? 'Loading devices...' : 'Select Device *' }}</option>
                <option v-for="device in stationDevices" :key="device.stationID" :value="device.customLabel1">
                  {{ device.customLabel1 }}
                </option>
              </select>
              </div>
            </div>
  
        <!-- Progress Border Container -->
        <div class="progress-border-container">
          <!-- Question Content -->
          <div 
            class="question-cloud"
            :style="getQuestionCloudStyle()"
          >
            <!-- Rank Tag Chip -->
            <div v-if="currentQuestion" class="rank-tag-chip" :class="getQuestionLevelClass()">
              {{ getRankDisplayText() }}
            </div>
            
            <!-- Dynamic Question Display -->
            <div v-if="currentQuestion" class="question-section">
              <div class="question-header">
                <h2 class="gentle-question" v-html="currentQuestion.text"></h2>
              </div>
              <p class="question-description">Please select the option that best describes your current situation</p>
              
              <div class="answers-container">
                <template v-for="answer in currentQuestion.answers">
                  <div 
                    :key="answer.answerID"
                    class="answer-option"
                    :class="{ 
                      'selected': selectedAnswers[currentQuestion.questionID] === answer.answerID,
                      [`rank-${answer.rank}`]: true,
                      ...getAnswerCSSClasses(answer)
                    }"
                    :style="getAnswerOptionStyle(answer)"
                    :title="selectedAnswers[currentQuestion.questionID] === answer.answerID ? 'Click again to deselect' : 'Click to select'"
                    @click="selectAnswer(currentQuestion.questionID, answer.answerID)"
                  >
                    <div class="answer-content">
                      <div class="answer-text">{{ answer.text }}</div>
                      <div class="answer-rank">Level {{ answer.rank }}</div>
                    </div>
                  </div>
                  
                  <!-- Continue button immediately after selected answer -->
                  <div 
                    v-if="selectedAnswers[currentQuestion.questionID] === answer.answerID && canProceed" 
                    :key="`continue-${answer.answerID}`"
                    class="continue-button-after-selected"
                  >
                    <button @click="nextStep" class="continue-button-dynamic">
                      <span>{{ getNextButtonText() }}</span>
                      <span class="nav-icon">🌟</span>
                    </button>
                  </div>
                </template>
              </div>
            </div>
          </div>
        </div>
  
        <!-- Supportive Navigation (for tablet and mobile) -->
        <div v-if="currentStep > 0 && !loading && !showResults" class="supportive-navigation">
          <button 
            @click="previousStep"
            class="nav-cloud prev-cloud"
          >
            <span class="nav-icon">🌙</span>
            <span>Previous</span>
          </button>
        </div>
        
        <!-- Encouragement Message -->
        <div class="encouragement-container">
          <div class="encouragement-message">{{ getEncouragementMessage() }}</div>
        </div>
        </div>
        </div>
      </div>
  
      <!-- Results/Thank You Page -->
      <div v-if="!loading && showResults" class="healing-results">
        <div class="results-sanctuary">
          <!-- Landing Page Content -->
          <div v-if="currentLandingPage" class="landing-page-content" v-html="currentLandingPage.html"></div>
          
          <!-- Default Landing Page Content -->
          <div v-else-if="surveyData.defaultLandingPageID && landingPages.length > 0" class="landing-page-content" v-html="getDefaultLandingPage()"></div>
          
          <!-- Default Results Content -->
          <div v-else>
            <div class="celebration-header">
              <div class="celebration-icon">💙</div>
              <h2 class="celebration-title">Assessment Complete</h2>
              <p class="celebration-message">Thank you for completing your assessment</p>
            </div>
            
            <div class="results-garden">
              <div class="level-badge" :class="getOverallLevelClass()">
                <div class="badge-icon">{{ getOverallLevelEmoji() }}</div>
                <div class="badge-text">{{ getOverallLevelText() }}</div>
              </div>
            </div>
            
            <div class="healing-message">
              <div class="message-icon">💙</div>
              <p>{{ getThankYouMessage() }}</p>
            </div>
            
            <!-- Submission Status -->
            <div class="submission-status">
              <div v-if="submittingSurvey" class="submission-loading">
                <div class="loading-spinner"></div>
                <p>Submitting your responses...</p>
              </div>
              <div v-else-if="surveySubmitted" class="submission-success">
                <div class="success-icon">✅</div>
                <p>Your responses have been submitted successfully!</p>
              </div>
              <div v-else class="submission-pending">
                <div class="pending-icon">⏳</div>
                <p>Preparing to submit your responses...</p>
              </div>
            </div>
          </div>
          
          <div class="support-actions">
            <button @click="resetAssessment" class="support-button restart-button">
              <span class="button-icon">🔄</span>
              <span>Take Assessment Again</span>
            </button>
            
            <!-- Manual submission button (only show if not submitted and not currently submitting) -->
            <button 
              v-if="!surveySubmitted && !submittingSurvey" 
              @click="submitSurveyData" 
              class="support-button submit-button"
              :disabled="submittingSurvey || !canSubmitSurvey"
              :title="!canSubmitSurvey ? 'Please select both Shift and Location to submit' : ''"
            >
              <span class="button-icon">📤</span>
              <span>{{ submittingSurvey ? 'Submitting...' : 'Submit Responses' }}</span>
            </button>
          </div>
          
          <div class="final-encouragement">
            <p>Remember: Your wellbeing matters. Take care of yourself. 💙</p>
          </div>
        </div>
      </div>
  
      <!-- Debug Button for Testing -->
      <div v-if="!loading && !showResults && questions.length > 0" class="debug-button-container">
        <div @click="showApiResponse" class="debug-button" title="Show API Response">
          <div class="code-icon-circle">
            <span class="code-brackets">{}</span>
          </div>
        </div>
      </div>
  
      <!-- Debug Modal -->
      <div v-if="showDebugModal" class="debug-modal" @click="closeDebugModal">
        <div class="debug-modal-content" @click.stop>
          <div class="debug-modal-header">
            <h3>Parsed API Response - Survey ID: {{ surveyId }}</h3>
            <div class="debug-header-controls">
              <button @click="copyJson" class="debug-copy-btn" title="Copy JSON">Copy</button>
              <button @click="closeDebugModal" class="debug-close-btn">×</button>
            </div>
          </div>
          <div class="debug-modal-body">
            <pre class="debug-json">{{ prettyApiResponse }}</pre>
          </div>
        </div>
      </div>
    </div>
  </template>
  
  <script>
  export default {
    name: 'StressThermometer',
    data() {
      return {
        loading: true,
        currentStep: 0,
        selectedAnswers: {},
        completedQuestions: new Set(), // Track questions that have been completed (answered + moved past)
        showResults: false,
        currentLandingPage: null,
        enhancedQuestionnaire: null, // Store the questionnaire with user selected answers
        questionnaireStartTime: null, // Track when questionnaire starts
        questionnaireEndTime: null, // Track when questionnaire ends
        surveyData: {
          surveyID: null,
          surveyName: 'Patient Assessment',
          bAcuitySurvey: false,
          status: true
        },
        questions: [],
        landingPages: [],
        edges: [],
        accessibleQuestions: [],
        surveyId: 6,
        showDebugModal: false,
        prettyApiResponse: null,
        rawApiResponse: null,
        initialized: false, // Add this to track if component has been initialized
        submittingSurvey: false, // Track survey submission status
        surveySubmitted: false, // Track if survey has been submitted
        selectedShift: '', // Track selected shift
        selectedLocation: '', // Track selected location (now device name)
        stationDevices: [], // Store station devices from API
        loadingDevices: false, // Track loading state for devices
        selectedDevice: null // Store the complete selected device object
      }
    },
    
    mounted() {
      // Auto-detect shift based on current time
      this.autoDetectShift();
      // Fetch station devices
      this.fetchStationDevices();
    },
    computed: {
      // Check if this is a conditional questionnaire
      isComplexQuestionnaire() {
        return this.edges.length > 0 || 
               this.landingPages.length > 0 || 
               this.surveyData.defaultLandingPageID !== undefined;
      },
      
      progressPercentage() {
        if (this.questions.length === 0) return 0;
        
        if (this.isComplexQuestionnaire) {
          // For conditional questionnaires, calculate based on current step position
          const totalAccessibleQuestions = this.accessibleQuestions.length;
          
          // Progress should reflect the current step (0-based) + 1 if current question is answered
          let currentProgress = this.currentStep;
          if (this.currentQuestion && this.selectedAnswers[this.currentQuestion.questionID]) {
            currentProgress += 1;
          }
          
          if (totalAccessibleQuestions === 0) return 0;
          return Math.min((currentProgress / totalAccessibleQuestions) * 100, 100);
        } else {
          // For simple questionnaires, calculate based on current step position
          const totalQuestions = this.questions.length;
          
          // Progress should reflect the current step (0-based) + 1 if current question is answered
          let currentProgress = this.currentStep;
          if (this.currentQuestion && this.selectedAnswers[this.currentQuestion.questionID]) {
            currentProgress += 1;
          }
          
          if (totalQuestions === 0) return 0;
          return Math.min((currentProgress / totalQuestions) * 100, 100);
        }
      },
      
      totalSteps() {
        if (this.isComplexQuestionnaire) {
          return this.accessibleQuestions.length;
        } else {
          return this.questions.length;
        }
      },
      
      currentQuestion() {
        if (this.isComplexQuestionnaire) {
          if (this.accessibleQuestions.length === 0 || this.currentStep >= this.accessibleQuestions.length) {
            return null;
          }
          return this.accessibleQuestions[this.currentStep];
        } else {
          // Simple questionnaire logic - unchanged
          if (this.questions.length === 0 || this.currentStep >= this.questions.length) {
            return null;
          }
          return this.questions[this.currentStep];
        }
      },
      
      canProceed() {
        if (!this.currentQuestion) return false;
        return this.selectedAnswers[this.currentQuestion.questionID] !== undefined;
      },
  
      // Detect if the survey is a stoplight survey
      isStoplightSurvey() {
        return this.surveyData.surveyID === 3 || this.surveyData.surveyCSS === 'stoplight-survey';
      },
  
      // Auto-initialization trigger
      autoInit() {
        if (!this.initialized) {
          console.log('Auto-initializing component...');
          this.initializeComponent();
        }
        return '';
      },
  
      // Check if survey can be submitted (both dropdowns filled)
      canSubmitSurvey() {
        return this.selectedShift && this.selectedLocation;
      }
    },
    
    methods: {
      // Auto-detect shift based on current system time
      autoDetectShift() {
        const currentHour = new Date().getHours();
        
        // Morning shift: 6 AM to 6 PM (6-17)
        // Night shift: 6 PM to 6 AM (18-5)
        if (currentHour >= 6 && currentHour < 18) {
          this.selectedShift = 'morning';
        } else {
          this.selectedShift = 'night';
        }
      },
      
      // Initialize the component - call this from parent component
      initializeComponent() {
        this.getSurveyIdFromURL();
        this.fetchSurveyData();
        this.initialized = true;
      },
      
      // Fetch station devices from API
      async fetchStationDevices() {
        try {
          this.loadingDevices = true;
          console.log('Fetching station devices...');
          
          const response = await fetch('https://demo.galaxycreations.com/gwas/api/ReportDefaults/rptStationDevices/25');
          
          if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
          }
          
          const devices = await response.json();
          
          // Filter out devices with empty customLabel1
          this.stationDevices = devices.filter(device => 
            device.customLabel1 && 
            device.customLabel1.trim() !== ''
          );
          
          console.log('Station devices loaded:', this.stationDevices);
          
        } catch (error) {
          console.error('Error fetching station devices:', error);
          
          // Fallback to original location options if API fails
          this.stationDevices = [
            {
              customLabel1: 'ICU',
              customField1: '192.168.1.10',
              stationID: '1020',
              stationNameCommon: 'ICU'
            }
          ];
          
          // Show error message to user
          this.$nextTick(() => {
            alert('Unable to load device list. Using default options.');
          });
        } finally {
          this.loadingDevices = false;
        }
      },
      
      // Handle device selection from dropdown
      onDeviceSelection() {
        if (this.selectedLocation) {
          // Find the selected device object
          this.selectedDevice = this.stationDevices.find(device => 
            device.customLabel1 === this.selectedLocation
          );
          console.log('Selected device:', this.selectedDevice);
        } else {
          this.selectedDevice = null;
        }
      },
      
      // Get survey ID from URL parameters
      getSurveyIdFromURL() {
        const urlParams = new URLSearchParams(window.location.search);
        const surveyIdParam = urlParams.get('surveyId') || urlParams.get('id');
        
        if (surveyIdParam) {
          this.surveyId = parseInt(surveyIdParam);
          console.log('Survey ID from URL:', this.surveyId);
        } else {
          console.log('No survey ID in URL, using default:', this.surveyId);
        }
      },
      
      async fetchSurveyData() {
        try {
          this.loading = true;
          const response = await fetch(`https://demo.galaxycreations.com/gwas/api/Survey/GetSurveyFlow/${this.surveyId}/0`);
          
          if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
          }
          
          const data = await response.json();
          
          // Store raw API response for debugging
          this.rawApiResponse = data;
          
          // Check if the response is empty or contains no valid data
          if (!data || data.length === 0 || data[0] === "{}" || data[0] === "null") {
            throw new Error('No survey data found for this ID');
          }
          
          // Parse the JSON string from the response
          const surveyData = JSON.parse(data[0]);
          
          // Validate that we have the required survey structure
          if (!surveyData.survey || !surveyData.nodes || surveyData.nodes.length === 0) {
            throw new Error('Invalid survey data structure');
          }
          
          this.surveyData = surveyData.survey;
          this.questions = surveyData.nodes.sort((a, b) => a.rank - b.rank);
          
          // Load conditional questionnaire data if present
          this.landingPages = surveyData.landingPages || [];
          this.edges = surveyData.edges || [];
          
          // Initialize accessible questions for conditional questionnaires
          if (this.isComplexQuestionnaire) {
            this.initializeComplexQuestionnaire();
          }
          
          console.log('Survey data loaded:', this.surveyData);
          console.log('Questions loaded:', this.questions.length);
          console.log('Is complex questionnaire:', this.isComplexQuestionnaire);
          
          // Validate that we have questions
          if (!this.questions || this.questions.length === 0) {
            throw new Error('No questions found in the survey data');
          }
          
          // Start the questionnaire timer
          this.questionnaireStartTime = new Date();
          console.log('Questionnaire started at:', this.questionnaireStartTime);
          
        } catch (error) {
          console.error('Error fetching survey data:', error);
          
          // Set error state
          this.surveyData = {
            surveyID: null,
            surveyName: 'Survey Not Found',
            bAcuitySurvey: false,
            status: false
          };
          this.questions = [];
          this.landingPages = [];
          this.edges = [];
          this.accessibleQuestions = [];
          
          // Show error message to user
          this.$nextTick(() => {
            if (error.message === 'No survey data found for this ID') {
              alert(`Survey with ID ${this.surveyId} not found. Please try a different survey ID.`);
            } else {
              alert('Unable to load assessment questions. Please try refreshing the page or contact support if the problem persists.');
            }
          });
        } finally {
          this.loading = false;
        }
      },
      
      // Initialize complex questionnaire logic
      initializeComplexQuestionnaire() {
        // Start with the first question (rank 1)
        const firstQuestion = this.questions.find(q => q.rank === 1);
        if (firstQuestion) {
          this.accessibleQuestions = [firstQuestion];
          this.currentStep = 0;
          this.completedQuestions.clear(); // Reset completed questions when initializing
        }
      },
      
      // Get next question ID for conditional questionnaires
      getNextQuestionID(questionID, answerID) {
        // First check if there's a direct nextQuestionID in the answer
        const question = this.questions.find(q => q.questionID === questionID);
        if (question) {
          const answer = question.answers.find(a => a.answerID === answerID);
          if (answer && answer.nextQuestionID && answer.nextQuestionID !== -1) {
            return answer.nextQuestionID;
          }
        }
        
        // Then check edges for conditional logic
        const edge = this.edges.find(e => 
          e.fromQuestionID === questionID && e.viaAnswerID === answerID
        );
        if (edge && edge.toQuestionID !== -1) {
          return edge.toQuestionID;
        }
        
        // Check for default next question
        if (question && question.defaultNextQuestionID && question.defaultNextQuestionID !== -1) {
          return question.defaultNextQuestionID;
        }
        
        return null; // No next question
      },
      
      // Handle landing page navigation
      handleLandingPageNavigation(questionID, answerID) {
        const question = this.questions.find(q => q.questionID === questionID);
        if (question) {
          const answer = question.answers.find(a => a.answerID === answerID);
          if (answer && answer.toLandingPageID && answer.toLandingPageID !== -1) {
            const landingPage = this.landingPages.find(lp => lp.landingPageID === answer.toLandingPageID);
            if (landingPage) {
              this.currentLandingPage = landingPage;
              return true;
            }
          }
        }
        return false;
      },
      
      selectAnswer(questionId, answerId) {
        // If the same answer is already selected, deselect it
        if (this.selectedAnswers[questionId] === answerId) {
          this.$delete(this.selectedAnswers, questionId);
          console.log('Deselected answer for question:', questionId);
        } else {
          // Select the new answer
          this.$set(this.selectedAnswers, questionId, answerId);
          console.log('Selected answer for question:', questionId, answerId);
        }
        
        // If this question was previously completed, remove it from completed set
        // since the user is changing their answer
        if (this.completedQuestions.has(questionId)) {
          this.completedQuestions.delete(questionId);
          console.log('Removed question from completed due to answer change:', questionId);
        }
        
        // Handle conditional logic if this is a complex questionnaire
        if (this.isComplexQuestionnaire) {
          if (this.selectedAnswers[questionId]) {
            // Answer was selected, handle conditional logic
            this.handleConditionalLogic(questionId, answerId);
          } else {
            // Answer was deselected, handle deselection logic
            this.handleDeselectionLogic(questionId);
          }
          
          // After handling conditional logic, ensure currentStep is within bounds
          // This is crucial if the new path is shorter than the previous one.
          if (this.currentStep >= this.accessibleQuestions.length) {
            this.currentStep = Math.max(0, this.accessibleQuestions.length - 1);
          }
        }
      },
      
      // Handle conditional logic for complex questionnaires
      handleConditionalLogic(questionId, answerId) {
        // Check for landing page navigation first
        if (this.handleLandingPageNavigation(questionId, answerId)) {
          // If it leads to a landing page, we don't add a next question to accessibleQuestions
          // We just ensure the current path is correctly truncated.
          const currentQuestionIndex = this.accessibleQuestions.findIndex(q => q.questionID === questionId);
          if (currentQuestionIndex !== -1) {
            // Truncate accessibleQuestions to only include up to the current question
            this.accessibleQuestions = this.accessibleQuestions.slice(0, currentQuestionIndex + 1);
          }
          // The landing page will be shown when they click Continue and reach the end
          return;
        }
        
        // Get next question ID for reference (but don't auto-advance)
        const nextQuestionID = this.getNextQuestionID(questionId, answerId);
        
        // Find the index of the current question in the accessibleQuestions array
        const currentQuestionIndex = this.accessibleQuestions.findIndex(q => q.questionID === questionId);
  
        if (currentQuestionIndex !== -1) {
          // Truncate accessibleQuestions to only include up to the current question
          // This clears any subsequent questions from a previous path
          this.accessibleQuestions = this.accessibleQuestions.slice(0, currentQuestionIndex + 1);
        }
  
        if (nextQuestionID) {
          // Find the next question
          const nextQuestion = this.questions.find(q => q.questionID === nextQuestionID);
          if (nextQuestion) {
            // Add to accessible questions if not already there
            if (!this.accessibleQuestions.find(q => q.questionID === nextQuestionID)) {
              this.accessibleQuestions.push(nextQuestion);
            }
          }
        } else {
          // If there's no nextQuestionID, it means this path ends here (or leads to a landing page already handled)
          // Ensure no further questions are in accessibleQuestions beyond the current one.
          if (currentQuestionIndex !== -1) {
            this.accessibleQuestions = this.accessibleQuestions.slice(0, currentQuestionIndex + 1);
          }
        }
      },
      
      // Handle deselection logic for complex questionnaires
      handleDeselectionLogic(questionId) {
        // Find the index of the current question in the accessibleQuestions array
        const currentQuestionIndex = this.accessibleQuestions.findIndex(q => q.questionID === questionId);
        
        if (currentQuestionIndex !== -1) {
          // Truncate accessibleQuestions to only include up to the current question
          // This removes any subsequent questions that were added based on the deselected answer
          this.accessibleQuestions = this.accessibleQuestions.slice(0, currentQuestionIndex + 1);
          console.log('Truncated accessible questions after deselection:', this.accessibleQuestions.length);
        }
      },
      
      nextStep() {
        console.log('nextStep called - isComplexQuestionnaire:', this.isComplexQuestionnaire);
        console.log('Current step:', this.currentStep);
        console.log('Accessible questions length:', this.accessibleQuestions.length);
        console.log('Questions length:', this.questions.length);
        
        // Mark current question as completed before moving to next step
        if (this.currentQuestion) {
          this.completedQuestions.add(this.currentQuestion.questionID);
          console.log('Marked question as completed:', this.currentQuestion.questionID);
        }
        
        if (this.isComplexQuestionnaire) {
          // For conditional questionnaires, check if there's a next question available
          if (this.currentStep < this.accessibleQuestions.length - 1) {
            console.log('Moving to next question in conditional questionnaire');
            this.currentStep++;
          } else {
            console.log('Reached end of accessible questions in conditional questionnaire');
            // Get the current question for processing
            const currentQuestion = this.accessibleQuestions[this.currentStep];
            
            // Check if there are more questions that could be unlocked
            if (currentQuestion) {
              console.log('Checking for more questions to unlock');
              const selectedAnswerId = this.selectedAnswers[currentQuestion.questionID];
              if (selectedAnswerId) {
                const nextQuestionID = this.getNextQuestionID(currentQuestion.questionID, selectedAnswerId);
                console.log('Next question ID:', nextQuestionID);
                if (nextQuestionID) {
                  const nextQuestion = this.questions.find(q => q.questionID === nextQuestionID);
                  if (nextQuestion && !this.accessibleQuestions.find(q => q.questionID === nextQuestionID)) {
                    console.log('Adding new question to accessible questions');
                    this.accessibleQuestions.push(nextQuestion);
                    this.currentStep++;
                    return;
                  } else {
                    console.log('Next question already exists or not found');
                  }
                } else {
                  console.log('No next question ID found');
                }
              } else {
                console.log('No selected answer for current question');
              }
            } else {
              console.log('No current question found');
            }
            
            // Check if current answer has a landing page
            if (currentQuestion) {
              const selectedAnswerId = this.selectedAnswers[currentQuestion.questionID];
              if (selectedAnswerId) {
                const selectedAnswer = currentQuestion.answers.find(a => a.answerID === selectedAnswerId);
                if (selectedAnswer && selectedAnswer.toLandingPageID && selectedAnswer.toLandingPageID !== -1) {
                  // Show landing page in results
                  const landingPage = this.landingPages.find(lp => lp.landingPageID === selectedAnswer.toLandingPageID);
                  if (landingPage) {
                    this.currentLandingPage = landingPage;
                  }
                }
              }
            }
            
            // Check if dropdowns are filled before showing results
            if (!this.canSubmitSurvey) {
              console.log('Cannot proceed to results - missing required fields (Shift/Location)');
              // Show validation alert and prevent progression
              const missingFields = [];
              if (!this.selectedShift) missingFields.push('Shift');
              if (!this.selectedLocation) missingFields.push('Location');
              alert(`Please select ${missingFields.join(' and ')} before completing the survey.`);
              return; // Stop progression to results page
            }
            
            // No more questions available, show results
            console.log('Reached end of conditional questionnaire - showing results');
            this.showResults = true;
            // Submit survey data when assessment is completed
            if (!this.surveySubmitted) {
              console.log('Calling submitSurveyData from conditional questionnaire');
              this.submitSurveyData();
            } else {
              console.log('Survey already submitted, skipping submission');
            }
          }
        } else {
          // Simple questionnaire logic - unchanged
          if (this.currentStep < this.questions.length - 1) {
            this.currentStep++;
          } else {
            // Check if dropdowns are filled before showing results
            if (!this.canSubmitSurvey) {
              console.log('Cannot proceed to results - missing required fields (Shift/Location)');
              // Show validation alert and prevent progression
              const missingFields = [];
              if (!this.selectedShift) missingFields.push('Shift');
              if (!this.selectedLocation) missingFields.push('Location');
              alert(`Please select ${missingFields.join(' and ')} before completing the survey.`);
              return; // Stop progression to results page
            }
            
            console.log('Reached end of simple questionnaire - showing results');
            this.showResults = true;
            // Submit survey data when assessment is completed
            if (!this.surveySubmitted) {
              console.log('Calling submitSurveyData from simple questionnaire');
              this.submitSurveyData();
            } else {
              console.log('Survey already submitted, skipping submission');
            }
          }
        }
      },
      
      previousStep() {
        if (this.currentStep > 0) {
          // Remove the current question from completed questions when going back
          if (this.currentQuestion) {
            this.completedQuestions.delete(this.currentQuestion.questionID);
            console.log('Removed question from completed:', this.currentQuestion.questionID);
          }
          this.currentStep--;
        }
      },
      
      getEncouragementMessage() {
        const messages = [
          "You're doing wonderfully 💫",
          "Thank you for your honesty 🌸",
          "Every step matters 🌟",
          "You're being so brave 💙",
          "Your feelings are valid 🤗",
          "Take your time, no rush 🌿"
        ];
        return messages[Math.floor(Math.random() * messages.length)];
      },
      
      getNextButtonText() {
        // Always show "Continue" for better UX
        return 'Continue';
      },
      
      getOverallLevelClass() {
        const averageRank = this.calculateAverageRank();
        if (averageRank <= 1.5) return 'green';
        if (averageRank <= 2.5) return 'yellow';
        if (averageRank <= 3.5) return 'orange';
        return 'red';
      },
      
      getOverallLevelEmoji() {
        const averageRank = this.calculateAverageRank();
        if (averageRank <= 1.5) return '✅';
        if (averageRank <= 2.5) return '⚠️';
        if (averageRank <= 3.5) return '🚨';
        return '🚨';
      },
      
      getOverallLevelText() {
        const averageRank = this.calculateAverageRank();
        if (averageRank <= 1.5) return 'Level I - Independent';
        if (averageRank <= 2.5) return 'Level II - Moderate Care';
        if (averageRank <= 3.5) return 'Level III - High Care';
        return 'Level IV - Critical Care';
      },
      
      calculateAverageRank() {
        const answeredQuestions = Object.keys(this.selectedAnswers);
        if (answeredQuestions.length === 0) return 0;
        
        let totalRank = 0;
        let questionCount = 0;
        
        answeredQuestions.forEach(questionId => {
          const question = this.questions.find(q => q.questionID === parseInt(questionId));
          if (question) {
            const answer = question.answers.find(a => a.answerID === this.selectedAnswers[questionId]);
            if (answer) {
              totalRank += answer.rank;
              questionCount++;
            }
          }
        });
        
        return questionCount > 0 ? totalRank / questionCount : 0;
      },
      
      getThankYouMessage() {
        const averageRank = this.calculateAverageRank();
        if (averageRank <= 1.5) {
          return 'Thank you for completing your assessment. You appear to be functioning well and independently. Continue to maintain your current level of wellness.';
        } else if (averageRank <= 2.5) {
          return 'Thank you for your assessment. You may benefit from some additional support and monitoring. Please reach out if you need assistance.';
        } else if (averageRank <= 3.5) {
          return 'Thank you for your assessment. Your responses indicate you may need significant support. Please consider speaking with a healthcare provider.';
        } else {
          return 'Thank you for your assessment. Your responses suggest you may need immediate medical attention. Please contact your healthcare provider right away.';
        }
      },
      
      resetAssessment() {
        this.currentStep = 0;
        this.selectedAnswers = {};
        this.completedQuestions.clear(); // Reset completed questions
        this.showResults = false;
        this.currentLandingPage = null;
        this.questionnaireStartTime = null;
        this.questionnaireEndTime = null;
        this.enhancedQuestionnaire = null;
        this.surveySubmitted = false; // Reset submission status
        this.submittingSurvey = false; // Reset submission loading state
        
        // Reset complex questionnaire state
        if (this.isComplexQuestionnaire) {
          this.initializeComplexQuestionnaire();
        }
      },
      
  
  
      // Get dynamic background color for question cloud based on selected answer
      getQuestionCloudStyle() {
        if (!this.currentQuestion) return {};
        
        const selectedAnswerId = this.selectedAnswers[this.currentQuestion.questionID];
        if (!selectedAnswerId) return {};
        
        const selectedAnswer = this.currentQuestion.answers.find(a => a.answerID === selectedAnswerId);
        if (!selectedAnswer) return {};
        
        // Only apply subtle accent if the selected answer has CSS
        if (selectedAnswer.css && selectedAnswer.css.trim()) {
          // Extract the background color from CSS classes
          const cssClasses = selectedAnswer.css.split(' ');
          let accentColor = '';
          
          // Find the background color class
          cssClasses.forEach(cls => {
            if (cls.includes('bg-')) {
              // Map CSS classes to accent colors
              if (cls.includes('bg-red')) accentColor = '#dc2626'; // red-600
              else if (cls.includes('bg-orange')) accentColor = '#f97316'; // orange-500
              else if (cls.includes('bg-yellow')) accentColor = '#eab308'; // yellow-400
              else if (cls.includes('bg-green')) accentColor = '#16a34a'; // green-600
            }
          });
          
          if (accentColor) {
            return {
              borderColor: accentColor,
              borderWidth: '3px',
              boxShadow: `0 0 20px ${accentColor}20`
            };
          }
        }
        
        // Return default styling if no CSS is provided
        return {};
      },
  
      // Get dynamic styling for answer options
      getAnswerOptionStyle(answer) {
        // If CSS is provided in the API response, don't apply rank-based styling
        if (answer.css && answer.css.trim()) {
          return {};
        }
        
        // Only apply rank-based colors if no CSS is provided
        const color = this.getColorForRank(answer.rank);
        if (!color) return {};
        
        return {
          borderColor: `${color}60`,
          background: `${color}10`
        };
      },
  
      // Parse CSS classes from API response
      getAnswerCSSClasses(answer) {
        if (!answer.css || !answer.css.trim()) {
          return {
            'default-white': true
          };
        }
        
        const cssClasses = {};
        const classes = answer.css.split(' ');
        
        classes.forEach(cls => {
          if (cls.trim()) {
            cssClasses[cls.trim()] = true;
          }
        });
        
        return cssClasses;
      },
  
      // Get color based on answer rank
      getColorForRank(rank) {
        const colorMap = {
          1: '#00b894', // Green - Independent
          2: '#fdcb6e', // Yellow - Moderate assistance
          3: '#e17055', // Orange - High assistance
          4: '#d63031', // Red - Critical care
          0: '#636e72'  // Gray - N/A
        };
        
        return colorMap[rank] || '#636e72';
      },
  
      // Get question level class for styling
      getQuestionLevelClass() {
        if (!this.currentQuestion) return '';
        
        const rank = this.currentQuestion.rank;
        
        if (rank === 0) return 'level-0';
        if (rank === 1) return 'level-1';
        if (rank === 2) return 'level-2';
        if (rank === 3) return 'level-3';
        if (rank === 4) return 'level-4';
        return 'level-0';
      },
  
      // Get display text for the rank tag
      getRankDisplayText() {
        if (!this.currentQuestion) return '';
        
        // Show question number (step + 1) instead of rank value
        return `${this.currentStep + 1}`;
      },
  
      // Get default landing page HTML
      getDefaultLandingPage() {
        const defaultLandingPage = this.landingPages.find(lp => lp.landingPageID === this.surveyData.defaultLandingPageID);
        if (defaultLandingPage) {
          return defaultLandingPage.html;
        }
        return ''; // Return empty string if no default landing page found
      },
  
      showApiResponse() {
        // Parse and format the raw API response
        if (this.rawApiResponse && this.rawApiResponse.length > 0) {
          try {
            const parsedData = JSON.parse(this.rawApiResponse[0]);
            this.prettyApiResponse = JSON.stringify(parsedData, null, 2);
          } catch (error) {
            this.prettyApiResponse = JSON.stringify(this.rawApiResponse, null, 2);
          }
        } else {
          this.prettyApiResponse = "No API response available";
        }
        this.showDebugModal = true;
      },
  
      closeDebugModal() {
        this.showDebugModal = false;
        this.prettyApiResponse = null;
      },
  
      copyJson() {
        if (this.prettyApiResponse) {
          navigator.clipboard.writeText(this.prettyApiResponse).then(() => {
            alert('JSON copied to clipboard!');
          }).catch(err => {
            console.error('Failed to copy JSON:', err);
            alert('Failed to copy JSON to clipboard.');
          });
        }
      },
  
      // Submit survey data to API
      async submitSurveyData() {
        console.log('submitSurveyData called - Survey ID:', this.selectedLocation, this.surveyData.surveyID || this.surveyId);
        console.log('Selected answers:', this.selectedAnswers);
        console.log('Is complex questionnaire:', this.isComplexQuestionnaire);
        
        // Validate that both dropdowns are filled
        if (!this.selectedShift || !this.selectedLocation) {
          const missingFields = [];
          if (!this.selectedShift) missingFields.push('Shift');
          if (!this.selectedLocation) missingFields.push('Location');
          
          alert(`Please select ${missingFields.join(' and ')} before submitting the survey.`);
          return; // Exit early if validation fails
        }
        
        try {
          this.submittingSurvey = true;
          
          // Record questionnaire end time
          this.questionnaireEndTime = new Date();
          console.log('Questionnaire ended at:', this.questionnaireEndTime);
          
          // Prepare survey answers in the required format: questionID::questionText::answerText::answerID::rank~
          const surveyAnswers = Object.keys(this.selectedAnswers)
            .map(questionId => {
              const answerId = this.selectedAnswers[questionId];
              const question = this.questions.find(q => q.questionID === parseInt(questionId));
              const answer = question ? question.answers.find(a => a.answerID === answerId) : null;
              
              if (question && answer) {
                // Format: questionID::questionText::answerText::answerID::rank
                return `rd${questionId}::${question.text}::${answer.text}::${answerId}::${answer.rank}`;
              }
              return null;
            })
            .filter(item => item !== null) // Remove any null items
            .join('~'); // Join with '~' separator
          
          // Calculate survey score as completion percentage (0-100)
          const answeredQuestionsForPayload = Object.keys(this.selectedAnswers).length;
          const totalQuestionsForPayload = this.questions.length;
          // Use actual questions shown to user for conditional questionnaires
          const questionsShownForPayload = this.isComplexQuestionnaire ? this.accessibleQuestions.length : totalQuestionsForPayload;
          const completionRateForPayload = questionsShownForPayload > 0 ? (answeredQuestionsForPayload / questionsShownForPayload) * 100 : 100;
          const surveyScore = Math.round(completionRateForPayload);
          const surveyTotalScore = Object.keys(this.selectedAnswers).length; // Total questions answered
          
          // Build enhanced questionnaire with user selected answers
          const enhancedQuestionnaire = this.buildEnhancedQuestionnaire();
          
          // Prepare the request payload with all required parameters
          const payload = {
            VisitId: "100164109",
            StationId: this.selectedDevice ? parseInt(this.selectedDevice.stationID) : 1020,
            BedId: 151,
            Shift: this.selectedShift, // Use the actual selected shift from dropdown
            SurveyId: this.surveyData.surveyID || this.surveyId,
            SurveyAnswers: surveyAnswers,
            SurveyScore: surveyScore,
            SurveyScoreId: 21,
            SurveyTotalScore: surveyTotalScore,
            StaffName: "",
            LoginName: this.selectedDevice ? this.selectedDevice.customField1 : "", // Use device IP address
            BAcuitySurvey: this.surveyData.bAcuitySurvey || false, // Use the actual bAcuitySurvey value from API response
            surveyAnswersJSON: enhancedQuestionnaire[0] // Add the enhanced questionnaire with user answers (extract string from array)
          };
          
          console.log('Submitting survey data:', payload);
          
          // Submit to original API
          const response = await fetch('https://demo.galaxycreations.com/gwas/api/Survey/ProcessSurvey', {
            method: 'POST',
            headers: {
              'Content-Type': 'application/json',
            },
            body: JSON.stringify(payload)
          });
          
          if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
          }
          
          // const result = await response.json();
          // console.log('Survey submission successful:', result);
          
          // Store the enhanced questionnaire for later access
          this.enhancedQuestionnaire = enhancedQuestionnaire;
          console.log('Enhanced questionnaire with user answers:', this.enhancedQuestionnaire);
          
          // Submit analytics data to AWS API Gateway
          await this.submitAnalyticsData();
          
          this.surveySubmitted = true;
          
          // Show success message to user
          this.$nextTick(() => {
            alert('Survey submitted successfully! Thank you for completing the assessment.');
          });
          
        } catch (error) {
          console.error('Error submitting survey data:', error);
          
          // Show error message to user
          this.$nextTick(() => {
            alert('Failed to submit survey data. Please try again or contact support if the problem persists.');
          });
        } finally {
          this.submittingSurvey = false;
        }
      },
  
      // Build enhanced questionnaire with user selected answers
      buildEnhancedQuestionnaire() {
        // Create a deep copy of the original questionnaire structure
        const enhancedQuestionnaireData = {
          survey: { ...this.surveyData },
          nodes: this.questions.map(question => {
            const enhancedNode = { ...question };
            
            // Add user selected answer information to each node
            const selectedAnswerId = this.selectedAnswers[question.questionID];
            if (selectedAnswerId) {
              const selectedAnswer = question.answers.find(a => a.answerID === selectedAnswerId);
              if (selectedAnswer) {
                enhancedNode.userSelectedAnswer = {
                  answerID: selectedAnswer.answerID,
                  text: selectedAnswer.text,
                  rank: selectedAnswer.rank,
                  selectedAt: new Date().toISOString()
                };
              }
            } else {
              // If no answer was selected, mark it as unanswered
              enhancedNode.userSelectedAnswer = null;
            }
            
            return enhancedNode;
          }),
          landingPages: [...(this.landingPages || [])],
          edges: [...(this.edges || [])]
        };
        
        // Return as string in array format (same as original API response)
        return [JSON.stringify(enhancedQuestionnaireData)];
      },
  
      // Get the enhanced questionnaire with user selected answers
      getEnhancedQuestionnaire() {
        return this.enhancedQuestionnaire;
      },
  
      // Calculate actual completion time in seconds
      getActualCompletionTimeInSeconds() {
        if (!this.questionnaireStartTime || !this.questionnaireEndTime) {
          return 0;
        }
        return Math.round((this.questionnaireEndTime - this.questionnaireStartTime) / 1000);
      },
  
      // Format actual completion time as "X minutes Y seconds"
      getActualCompletionTime() {
        const totalSeconds = this.getActualCompletionTimeInSeconds();
        if (totalSeconds === 0) {
          return "0 seconds";
        }
        
        const minutes = Math.floor(totalSeconds / 60);
        const seconds = totalSeconds % 60;
        
        if (minutes === 0) {
          return `${seconds} second${seconds !== 1 ? 's' : ''}`;
        } else if (seconds === 0) {
          return `${minutes} minute${minutes !== 1 ? 's' : ''}`;
        } else {
          return `${minutes} minute${minutes !== 1 ? 's' : ''} ${seconds} second${seconds !== 1 ? 's' : ''}`;
        }
      },
  
      // Build questionnaire path (Q1->Answer->Q2->Answer)
      buildQuestionnairePath() {
        const path = [];
        const sortedQuestions = this.questions.sort((a, b) => a.rank - b.rank);
        
        sortedQuestions.forEach(question => {
          const selectedAnswerId = this.selectedAnswers[question.questionID];
          if (selectedAnswerId) {
            const selectedAnswer = question.answers.find(a => a.answerID === selectedAnswerId);
            if (selectedAnswer) {
              path.push({
                questionID: question.questionID,
                questionText: question.text,
                questionRank: question.rank,
                answerID: selectedAnswer.answerID,
                answerText: selectedAnswer.text,
                answerRank: selectedAnswer.rank
              });
            }
          }
        });
        
        return path;
      },
  
      // Build detailed question analytics for QuickSight
      buildQuestionAnalytics() {
        const questionAnalytics = [];
        
        this.questions.forEach(question => {
          const selectedAnswerId = this.selectedAnswers[question.questionID];
          const isAnswered = !!selectedAnswerId;
          
          let selectedAnswer = null;
          if (isAnswered) {
            selectedAnswer = question.answers.find(a => a.answerID === selectedAnswerId);
          }
          
          questionAnalytics.push({
            questionID: question.questionID,
            questionText: question.text,
            questionType: question.type,
            questionRank: question.rank,
            isAnswered: isAnswered,
            selectedAnswerID: selectedAnswerId || null,
            selectedAnswerText: selectedAnswer ? selectedAnswer.text : null,
            selectedAnswerRank: selectedAnswer ? selectedAnswer.rank : null,
            totalAnswersAvailable: question.answers.length,
            answerOptions: question.answers.map(a => ({
              answerID: a.answerID,
              answerText: a.text,
              answerRank: a.rank
            }))
          });
        });
        
        return questionAnalytics;
      },
  
      // Submit comprehensive analytics data to AWS API Gateway for QuickSight (Questionnaire Level)
      async submitAnalyticsData() {
        try {
          console.log('Submitting analytics data to AWS API Gateway...');
          
          const timestamp = new Date().toISOString();
          const submissionDate = new Date().toISOString().split('T')[0]; // YYYY-MM-DD format
          const submissionTime = new Date().toLocaleTimeString('en-US', { hour12: false }); // HH:MM:SS format
          const submissionHour = new Date().getHours();
          const dayOfWeek = new Date().toLocaleDateString('en-US', { weekday: 'long' });
          const month = new Date().toLocaleDateString('en-US', { month: 'long' });
          const year = new Date().getFullYear();
          const submissionId = `${this.surveyData.surveyID || this.surveyId}_${Date.now()}`;
          
          // Calculate survey metrics
          const totalQuestions = this.questions.length;
          const answeredQuestions = Object.keys(this.selectedAnswers).length;
          
          // Calculate completion rate based on questions actually shown to user
          let completionRate;
          let questionsShownToUser;
          
          console.log('Debug - isComplexQuestionnaire:', this.isComplexQuestionnaire);
          console.log('Debug - totalQuestions:', totalQuestions);
          console.log('Debug - answeredQuestions:', answeredQuestions);
          console.log('Debug - accessibleQuestions.length:', this.accessibleQuestions.length);
          
          // Use actual questions shown to user (for conditional questionnaires)
          if (this.isComplexQuestionnaire) {
            questionsShownToUser = this.accessibleQuestions.length;
            completionRate = questionsShownToUser > 0 ? (answeredQuestions / questionsShownToUser) * 100 : 100;
            console.log(`Debug - Conditional: ${answeredQuestions}/${questionsShownToUser} = ${completionRate}%`);
          } else {
            // For simple questionnaires, use total questions
            questionsShownToUser = totalQuestions;
            completionRate = (answeredQuestions / totalQuestions) * 100;
            console.log(`Debug - Simple: ${answeredQuestions}/${questionsShownToUser} = ${completionRate}%`);
          }
          const averageRank = this.calculateAverageRank();
          
          // Calculate survey score as completion percentage (0-100)
          const surveyScore = Math.round(completionRate);
          
          // Calculate actual completion time
          const actualCompletionTimeInSeconds = this.getActualCompletionTimeInSeconds();
          const actualCompletionTimeFormatted = this.getActualCompletionTime();
          
          // Determine risk level based on average rank
  
          
          // Prepare question and answer summary data
          const answeredQuestionsList = this.prepareAnsweredQuestionsSummary();
          
          // Build comprehensive questionnaire data for QuickSight
          const questionnairePath = this.buildQuestionnairePath();
          const questionAnalytics = this.buildQuestionAnalytics();
          const enhancedQuestionnairePayload = this.buildEnhancedQuestionnaire();
          
          // Calculate answer distribution
          // const answerDistribution = this.calculateAnswerDistribution();
          
          // Calculate question category distribution
          
          // Prepare conditional questionnaire metadata
          const conditionalMetadata = this.prepareConditionalMetadata();
          
          // Create single analytics record for the entire questionnaire
          const analyticsRecord = {
            // Submission metadata
            submissionId: submissionId,
            timestamp: timestamp,
            submissionDate: submissionDate,
            submissionTime: submissionTime,
            submissionHour: submissionHour,
            dayOfWeek: dayOfWeek,
            month: month,
            year: year,
            
            // Survey information
            surveyId: this.surveyData.surveyID || this.surveyId,
            surveyName: this.surveyData.surveyName || 'Patient Assessment',
            AcuitySurvey: this.surveyData.bAcuitySurvey ? true : false,
            isComplexQuestionnaire: this.isComplexQuestionnaire,
            
            // Location and shift data
            deviceLocation: this.selectedLocation,
            deviceId: this.selectedDevice ? this.selectedDevice.stationID : null,
            deviceIP: this.selectedDevice ? this.selectedDevice.customField1 : null,
            shift: this.selectedShift,
            
            // Survey metrics
            totalQuestions: totalQuestions, // Total questions in the questionnaire
            questionsShownToUser: questionsShownToUser, // Questions actually shown to user (for conditional questionnaires)
            answeredQuestions: answeredQuestions, // Questions actually answered by user
            completionRate: Math.round(completionRate), // Completion rate based on questions shown to user
            // averageRank: Math.round(averageRank * 100) / 100,
            surveyScore: surveyScore,
            // riskLevel: riskLevel,
            // riskCategory: riskCategory,
            
  
  
            
            // Time-based analytics
            timeToComplete: actualCompletionTimeInSeconds, // Actual completion time in seconds
            timeToCompleteFormatted: actualCompletionTimeFormatted, // Actual completion time formatted (e.g., "1 minute 12 seconds")
            // estimatedTimeToComplete: this.getEstimatedCompletionTime(), // Removed - using actual time instead
            questionnaireStartTime: this.questionnaireStartTime ? this.questionnaireStartTime.toISOString() : null,
            questionnaireEndTime: this.questionnaireEndTime ? this.questionnaireEndTime.toISOString() : null,
            isWeekend: [0, 6].includes(new Date().getDay()),
            
            // Technical metadata
            userAgent: navigator.userAgent,
            screenResolution: `${screen.width}x${screen.height}`,
            deviceType: this.getDeviceType(),
            
            // Conditional questionnaire specific fields
  
            pathIdentifier: conditionalMetadata.pathIdentifier,
            // conditionalPathDescription: conditionalMetadata.pathDescription,
            averageQuestionDisplayTime: conditionalMetadata.averageQuestionDisplayTime,
            
            // Summary fields for answered questions (JSON format for flexibility)
            answeredQuestionsData: JSON.stringify(answeredQuestionsList),
            
            // Comprehensive questionnaire data for QuickSight analysis
            questionnairePath: JSON.stringify(questionnairePath), // Q1->Answer->Q2->Answer path
            questionAnalytics: JSON.stringify(questionAnalytics), // Detailed question-level data
            enhancedQuestionnaireData: enhancedQuestionnairePayload[0], // Complete questionnaire with user answers (already stringified)
            
            // Questionnaire structure data
            questionnaireStructure: JSON.stringify({
              survey: this.surveyData,
              totalQuestions: totalQuestions,
              questionTypes: this.questions.map(q => q.type),
              questionRanks: this.questions.map(q => q.rank),
              hasConditionalLogic: this.isComplexQuestionnaire,
              landingPagesCount: this.landingPages ? this.landingPages.length : 0,
              edgesCount: this.edges ? this.edges.length : 0
            }),
            
            // High-level insights
  
          };
          
          console.log('Analytics record for questionnaire:', analyticsRecord);
          
          // Submit to AWS API Gateway
          const analyticsResponse = await fetch('https://8s5zhkyjbh.execute-api.us-west-2.amazonaws.com/prod/survey-analytics', {
            method: 'POST',
            headers: {
              'Content-Type': 'application/json',
              'X-API-Key': 'your-api-key-here', // Replace with your actual API key if needed
            },
            body: JSON.stringify(analyticsRecord)
          });
          
          if (analyticsResponse.ok) {
            console.log('Analytics data submitted successfully to AWS');
          } else {
            console.warn('Analytics submission failed:', analyticsResponse.status, analyticsResponse.statusText);
            // Don't throw error to avoid breaking the main submission flow
          }
          
        } catch (error) {
          console.error('Error submitting analytics data:', error);
          // Don't throw error to avoid breaking the main submission flow
        }
      },
  
      // Prepare answered questions summary for questionnaire-level analytics
      prepareAnsweredQuestionsSummary() {
        const summary = [];
        
        Object.keys(this.selectedAnswers).forEach(questionId => {
          const answerId = this.selectedAnswers[questionId];
          const question = this.questions.find(q => q.questionID === parseInt(questionId));
          const answer = question ? question.answers.find(a => a.answerID === answerId) : null;
          
          if (question && answer) {
            summary.push({
              questionId: parseInt(questionId),
              questionText: question.text.replace(/<[^>]*>/g, ''), // Strip HTML tags
              questionCategory: this.getQuestionCategory(question.text),
              answerId: answerId,
              answerText: answer.text,
              answerRank: answer.rank,
              riskIndicator: answer.rank >= 3 ? 'High Risk' : answer.rank >= 2 ? 'Medium Risk' : 'Low Risk'
            });
          }
        });
        
        return summary;
      },
  
      // Calculate answer distribution for questionnaire
      calculateAnswerDistribution() {
        const distribution = {
          rank1Count: 0,
          rank2Count: 0,
          rank3Count: 0,
          rank4Count: 0,
          lowRiskCount: 0,
          mediumRiskCount: 0,
          highRiskCount: 0
        };
        
        Object.keys(this.selectedAnswers).forEach(questionId => {
          const answerId = this.selectedAnswers[questionId];
          const question = this.questions.find(q => q.questionID === parseInt(questionId));
          const answer = question ? question.answers.find(a => a.answerID === answerId) : null;
          
          if (answer) {
            // Count by rank
            if (answer.rank === 1) distribution.rank1Count++;
            else if (answer.rank === 2) distribution.rank2Count++;
            else if (answer.rank === 3) distribution.rank3Count++;
            else if (answer.rank === 4) distribution.rank4Count++;
            
            // Count by risk level
            if (answer.rank <= 1) distribution.lowRiskCount++;
            else if (answer.rank === 2) distribution.mediumRiskCount++;
            else if (answer.rank >= 3) distribution.highRiskCount++;
          }
        });
        
        return distribution;
      },
  
      // Calculate category distribution for questionnaire
      calculateCategoryDistribution() {
        const distribution = {
          painManagementCount: 0,
          mobilityCount: 0,
          mentalHealthCount: 0,
          medicationCount: 0,
          sleepRestCount: 0,
          nutritionCount: 0,
          careNeedsCount: 0,
          generalAssessmentCount: 0,
          categoriesAnswered: []
        };
        
        Object.keys(this.selectedAnswers).forEach(questionId => {
          const question = this.questions.find(q => q.questionID === parseInt(questionId));
          
          if (question) {
            const category = this.getQuestionCategory(question.text);
            
            // Count by category
            switch (category) {
              case 'Pain Management':
                distribution.painManagementCount++;
                break;
              case 'Mobility':
                distribution.mobilityCount++;
                break;
              case 'Mental Health':
                distribution.mentalHealthCount++;
                break;
              case 'Medication':
                distribution.medicationCount++;
                break;
              case 'Sleep & Rest':
                distribution.sleepRestCount++;
                break;
              case 'Nutrition':
                distribution.nutritionCount++;
                break;
              case 'Care Needs':
                distribution.careNeedsCount++;
                break;
              default:
                distribution.generalAssessmentCount++;
                break;
            }
            
            // Track unique categories answered
            if (!distribution.categoriesAnswered.includes(category)) {
              distribution.categoriesAnswered.push(category);
            }
          }
        });
        
        return distribution;
      },
  
      // Prepare conditional questionnaire metadata
      prepareConditionalMetadata() {
        if (!this.isComplexQuestionnaire) {
          return {
            isConditionalPath: false,
            questionsAccessedCount: 0,
            pathIdentifier: null,
            pathDescription: null,
            averageQuestionDisplayTime: null
          };
        }
        
        const questionsAccessedCount = this.accessibleQuestions.length;
        const answeredQuestionsInPath = this.accessibleQuestions.filter(q => 
          this.selectedAnswers[q.questionID]
        );
        
        // Generate path description
        let pathDescription = 'Conditional path: ';
        answeredQuestionsInPath.forEach((question, index) => {
          const answer = question.answers.find(a => a.answerID === this.selectedAnswers[question.questionID]);
          if (answer) {
            pathDescription += `Q${question.questionID}(${answer.text})`;
            if (index < answeredQuestionsInPath.length - 1) {
              pathDescription += ' → ';
            }
          }
        });
        
        // Generate clean, readable path identifier
        const pathIdentifier = `Survey: ${this.surveyData.surveyName || 'Survey'}\n${answeredQuestionsInPath.map(q => {
          const answer = q.answers.find(a => a.answerID === this.selectedAnswers[q.questionID]);
          const answerText = answer ? answer.text : 'No Answer';
          return `Q${q.questionID}(${q.text}) → ${answerText}`;
        }).join('\n')}`;
        
        // Estimate average question display time
        const averageQuestionDisplayTime = answeredQuestionsInPath.reduce((total, question) => {
          return total + this.estimateQuestionDisplayTime(question);
        }, 0) / Math.max(answeredQuestionsInPath.length, 1);
        
        return {
          isConditionalPath: true,
          questionsAccessedCount: questionsAccessedCount,
          pathIdentifier: pathIdentifier,
          pathDescription: pathDescription,
          averageQuestionDisplayTime: Math.round(averageQuestionDisplayTime)
        };
      },
  
      // Get dominant risk level from distribution
      getDominantRiskLevel(distribution) {
        const { lowRiskCount, mediumRiskCount, highRiskCount } = distribution;
        
        if (highRiskCount > mediumRiskCount && highRiskCount > lowRiskCount) {
          return 'High Risk';
        } else if (mediumRiskCount > lowRiskCount && mediumRiskCount > highRiskCount) {
          return 'Medium Risk';
        } else if (lowRiskCount > 0) {
          return 'Low Risk';
        } else {
          return 'Unknown';
        }
      },
  
      // Get primary concern category from distribution
      getPrimaryConcernCategory(distribution) {
        const categories = [
          { name: 'Pain Management', count: distribution.painManagementCount },
          { name: 'Mobility', count: distribution.mobilityCount },
          { name: 'Mental Health', count: distribution.mentalHealthCount },
          { name: 'Medication', count: distribution.medicationCount },
          { name: 'Sleep & Rest', count: distribution.sleepRestCount },
          { name: 'Nutrition', count: distribution.nutritionCount },
          { name: 'Care Needs', count: distribution.careNeedsCount },
          { name: 'General Assessment', count: distribution.generalAssessmentCount }
        ];
        
        const maxCategory = categories.reduce((max, current) => 
          current.count > max.count ? current : max
        );
        
        return maxCategory.count > 0 ? maxCategory.name : 'None';
      },
  
      // Estimate question display time (for analytics)
      estimateQuestionDisplayTime(question) {
        // Simple estimation based on question text length and complexity
        const baseTime = 5; // 5 seconds base time
        const textLength = question.text.length;
        const complexityFactor = question.answers ? question.answers.length * 2 : 5;
        return baseTime + Math.round(textLength / 10) + complexityFactor;
      },
  
      // Estimate answer selection time (for analytics)
      estimateAnswerSelectionTime(answer) {
        // Simple estimation based on answer text length and rank
        const baseTime = 3; // 3 seconds base time
        const textLength = answer.text.length;
        const complexityFactor = answer.rank * 2; // Higher ranks might require more thinking
        return baseTime + Math.round(textLength / 15) + complexityFactor;
      },
  
      // Helper method to categorize questions
      getQuestionCategory(questionText) {
        const text = questionText.toLowerCase();
        
        if (text.includes('pain') || text.includes('discomfort') || text.includes('hurt')) {
          return 'Pain Management';
        } else if (text.includes('mobility') || text.includes('walk') || text.includes('move')) {
          return 'Mobility';
        } else if (text.includes('mental') || text.includes('mood') || text.includes('stress') || text.includes('anxiety')) {
          return 'Mental Health';
        } else if (text.includes('medication') || text.includes('medicine') || text.includes('drug')) {
          return 'Medication';
        } else if (text.includes('sleep') || text.includes('rest') || text.includes('tired')) {
          return 'Sleep & Rest';
        } else if (text.includes('eat') || text.includes('appetite') || text.includes('food')) {
          return 'Nutrition';
        } else if (text.includes('care') || text.includes('help') || text.includes('assist')) {
          return 'Care Needs';
        } else {
          return 'General Assessment';
        }
      },
  
      // Helper method to estimate completion time
      getEstimatedCompletionTime() {
        // Simple estimation based on number of questions
        const baseTimePerQuestion = 30; // 30 seconds per question
        const totalQuestions = this.questions.length;
        return totalQuestions * baseTimePerQuestion;
      },
  
      // Helper method to detect device type
      getDeviceType() {
        const userAgent = navigator.userAgent.toLowerCase();
        if (/tablet|ipad|playbook|silk/.test(userAgent)) {
          return 'Tablet';
        } else if (/mobile|iphone|ipod|android|blackberry|opera|mini|windows\sce|palm|smartphone|iemobile/.test(userAgent)) {
          return 'Mobile';
        } else {
          return 'Desktop';
        }
      }
    }
  }
  </script>
  
  <style scoped>
  /* Import the same styles as App.vue */
  
  /* Ensure full width coverage */
  :deep(body), :deep(html) {
    margin: 0;
    padding: 0;
    width: 100%;
    overflow-x: hidden;
  }
  
  /* Mobile-specific full width fixes */
  @media (max-width: 767px) {
    .gentle-question,
    .question-description,
    .question-header,
    .question-section {
      width: 100% !important;
      max-width: 100% !important;
      display: block !important;
    }
  }
  .therapeutic-container {
    min-height: 100vh;
    width: 100%;
    max-width: 100%;
    margin: 0;
    padding: 0;
    background: linear-gradient(135deg, #ffeaa7 0%, #fab1a0 25%, #fd79a8 50%, #a29bfe 75%, #6c5ce7 100%);
    background-size: 400% 400%;
    animation: gentleGradient 20s ease infinite;
    font-family: 'Inter', 'Segoe UI', sans-serif;
    position: relative;
    overflow-x: hidden;
    box-sizing: border-box;
    display: block;
  }
  
  @keyframes gentleGradient {
    0% { background-position: 0% 50%; }
    50% { background-position: 100% 50%; }
    100% { background-position: 0% 50%; }
  }
  
  .floating-elements {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 1;
  }
  
  .floating-circle {
    position: absolute;
    border-radius: 50%;
    background: rgba(255, 255, 255, 0.1);
    animation: gentleFloat 15s ease-in-out infinite;
  }
  
  .circle-1 {
    width: 100px;
    height: 100px;
    top: 20%;
    left: 10%;
    animation-delay: 0s;
  }
  
  .circle-2 {
    width: 150px;
    height: 150px;
    top: 60%;
    right: 15%;
    animation-delay: 5s;
  }
  
  .circle-3 {
    width: 80px;
    height: 80px;
    bottom: 20%;
    left: 20%;
    animation-delay: 10s;
  }
  
  @keyframes gentleFloat {
    0%, 100% { transform: translateY(0px) rotate(0deg); }
    33% { transform: translateY(-20px) rotate(120deg); }
    66% { transform: translateY(10px) rotate(240deg); }
  }
  
  /* Header Styles */
  .gentle-header {
    max-width: 100%;
    margin: 0 auto;
    padding: 10px 20px;
    text-align: center;
    position: relative;
    z-index: 2;
    width: 100%;
    box-sizing: border-box;
    overflow-wrap: break-word;
    word-wrap: break-word;
  }
  
  .breathing-icon {
    margin-bottom: 5px;
  }
  
  .breath-circle {
    width: 80px;
    height: 80px;
    border: 4px solid rgba(255, 255, 255, 0.9);
    border-radius: 50%;
    margin: 0 auto;
    animation: breathe 4s ease-in-out infinite;
    box-shadow: 
      0 0 30px rgba(255, 255, 255, 0.5),
      0 0 60px rgba(116, 185, 255, 0.3),
      inset 0 0 30px rgba(255, 255, 255, 0.2);
    position: relative;
    background: linear-gradient(135deg, rgba(255, 255, 255, 0.1), rgba(255, 255, 255, 0.05));
    backdrop-filter: blur(10px);
  }
  
  .breath-circle::before {
    content: '';
    position: absolute;
    top: -6px;
    left: -6px;
    right: -6px;
    bottom: -6px;
    border: 1px solid rgba(255, 255, 255, 0.3);
    border-radius: 50%;
    animation: breathe 4s ease-in-out infinite reverse;
  }
  
  @keyframes breathe {
    0%, 100% { 
      transform: scale(1); 
      opacity: 0.8; 
      box-shadow: 
        0 0 20px rgba(255, 255, 255, 0.3),
        inset 0 0 20px rgba(255, 255, 255, 0.1);
    }
    50% { 
      transform: scale(1.1); 
      opacity: 1; 
      box-shadow: 
        0 0 30px rgba(255, 255, 255, 0.5),
        inset 0 0 30px rgba(255, 255, 255, 0.2);
    }
  }
  
  .caring-title {
    font-size: 2.8rem;
    color: white;
    margin-bottom: 10px;
    font-weight: 400;
    text-shadow: 
      0 4px 20px rgba(0,0,0,0.3),
      0 2px 10px rgba(0,0,0,0.2);
    letter-spacing: -0.5px;
    line-height: 1.2;
  }
  
  .supportive-message {
    font-size: 1.15rem;
    color: rgba(255, 255, 255, 0.9);
    margin-bottom: 5px;
    line-height: 1.6;
  }
  
  .encouragement {
    font-size: 1rem;
    color: rgba(255, 255, 255, 0.8);
    margin-bottom: 5px;
    font-style: italic;
  }
  
  /* Progress Section */
  .progress-section {
    margin-top: 5px;
    background: rgba(255, 255, 255, 0.1);
    backdrop-filter: blur(10px);
    border-radius: 15px;
    padding: 15px;
    border: 1px solid rgba(255, 255, 255, 0.2);
    max-width: 500px;
    margin-left: auto;
    margin-right: auto;
    width: 100%;
    box-sizing: border-box;
  }
  
  .progress-bar-container {
    margin-bottom: 10px;
  }
  
  .progress-bar-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 8px;
  }
  
  .progress-text {
    color: rgba(255, 255, 255, 0.9);
    font-size: 1rem;
    font-weight: 500;
  }
  
  .step-counter {
    color: rgba(255, 255, 255, 0.8);
    font-size: 0.9rem;
    background: rgba(255, 255, 255, 0.1);
    padding: 4px 12px;
    border-radius: 12px;
  }
  
  .progress-bar-track {
    position: relative;
    width: 100%;
    height: 16px;
    background: linear-gradient(135deg, rgba(255, 255, 255, 0.2), rgba(255, 255, 255, 0.1));
    border-radius: 12px;
    overflow: hidden;
    box-shadow: 
      inset 0 3px 8px rgba(0, 0, 0, 0.15),
      0 2px 4px rgba(255, 255, 255, 0.3),
      0 0 0 1px rgba(255, 255, 255, 0.3);
    border: 1px solid rgba(255, 255, 255, 0.4);
    backdrop-filter: blur(10px);
  }
  
  .progress-bar-fill {
    position: relative;
    height: 100%;
    background: linear-gradient(90deg, #667eea 0%, #764ba2 25%, #f093fb 50%, #f5576c 75%, #4facfe 100%);
    border-radius: 12px;
    transition: width 0.8s cubic-bezier(0.4, 0, 0.2, 1);
    overflow: hidden;
    box-shadow: 
      0 4px 15px rgba(102, 126, 234, 0.5),
      0 0 20px rgba(116, 185, 255, 0.3),
      inset 0 1px 0 rgba(255, 255, 255, 0.4);
  }
  
  .progress-bar-glow {
    position: absolute;
    top: 0;
    right: 0;
    width: 30px;
    height: 100%;
    background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.4));
    animation: progressGlow 2s ease-in-out infinite;
  }
  
  @keyframes progressGlow {
    0%, 100% { opacity: 0.4; }
    50% { opacity: 0.8; }
  }
  
  /* Loading State */
  .loading-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 100vh;
    color: white;
    text-align: center;
    background: linear-gradient(135deg, #ffeaa7 0%, #fab1a0 25%, #fd79a8 50%, #a29bfe 75%, #6c5ce7 100%);
    background-size: 400% 400%;
    animation: gentleGradient 20s ease infinite;
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    z-index: 1000;
  }
  
  .loading-dots {
    display: flex;
    gap: 10px;
    margin-bottom: 20px;
  }
  
  .dot {
    width: 15px;
    height: 15px;
    background-color: white;
    border-radius: 50%;
    animation: dot-pulse 1.5s infinite ease-in-out;
  }
  
  .dot:nth-child(1) { animation-delay: -0.32s; }
  .dot:nth-child(2) { animation-delay: -0.16s; }
  .dot:nth-child(3) { animation-delay: 0s; }
  
  @keyframes dot-pulse {
    0%, 80%, 100% { transform: translateY(0); opacity: 0.4; }
    40% { transform: translateY(-10px); opacity: 1; }
  }
  
  .loading-title {
    font-size: 1.8rem;
    color: white;
    margin-bottom: 15px;
    font-weight: 300;
    text-shadow: 0 2px 10px rgba(0,0,0,0.1);
  }
  
  .loading-text {
    font-size: 1.2rem;
    color: rgba(255, 255, 255, 0.9);
    font-weight: 500;
  }
  
  /* Error State */
  .error-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 60vh;
    color: white;
    text-align: center;
    padding: 20px;
  }
  
  .error-icon {
    font-size: 4rem;
    margin-bottom: 20px;
    animation: gentleBounce 2s ease-in-out infinite;
  }
  
  .error-title {
    font-size: 2rem;
    color: white;
    margin-bottom: 15px;
    font-weight: 300;
  }
  
  .error-message {
    font-size: 1.1rem;
    color: rgba(255, 255, 255, 0.9);
    margin-bottom: 30px;
    max-width: 500px;
    line-height: 1.6;
  }
  
  .retry-button {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 15px 25px;
    border: none;
    border-radius: 25px;
    font-size: 1rem;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.4s ease;
    background: linear-gradient(135deg, #74b9ff, #0984e3);
    color: white;
    box-shadow: 0 5px 15px rgba(0,0,0,0.2);
  }
  
  .retry-button:hover {
    transform: translateY(-2px);
    box-shadow: 0 10px 25px rgba(0,0,0,0.3);
  }
  
  /* Icon Styles */
  .nav-icon {
    font-size: 1.2rem;
  }
  
  .button-icon {
    font-size: 1.1rem;
  }
  
  /* Responsive Icon Sizes */
  @media (max-width: 1199px) {
    .nav-icon {
      font-size: 1rem;
    }
  }
  
  @media (max-width: 767px) {
    .nav-icon {
      font-size: 0.9rem;
    }
  }
  
  @media (max-width: 575px) {
    .nav-icon {
      font-size: 0.8rem;
    }
  }
  
  @media (max-width: 479px) {
    .nav-icon {
      font-size: 0.7rem;
    }
  }
  
  /* Question Container */
  .question-sanctuary {
    max-width: 600px;
    margin: 0 auto 20px;
    position: relative;
    z-index: 2;
  }
  
  .progress-border-container {
    position: relative;
    margin: 10px;
  }
  
  .question-cloud {
    background: linear-gradient(135deg, rgba(255, 255, 255, 0.98), rgba(255, 255, 255, 0.95));
    backdrop-filter: blur(20px);
    border-radius: 25px;
    padding: 15px;
    box-shadow: 
      0 25px 50px rgba(0, 0, 0, 0.12),
      0 0 0 1px rgba(255, 255, 255, 0.3),
      inset 0 1px 0 rgba(255, 255, 255, 0.4);
    border: 2px solid rgba(255, 255, 255, 0.4);
    min-height: 300px;
    display: flex;
    flex-direction: column;
    position: relative;
    transition: all 0.3s ease;
    z-index: 2;
  }
  
  /* Desktop/Laptop - Previous Button in Page Top Left */
  @media (min-width: 1200px) {
    .previous-button-responsive {
      position: fixed;
      top: 20px;
      left: 20px;
      display: flex;
      align-items: center;
      gap: 6px;
      padding: 6px 12px;
      border: none;
      border-radius: 18px;
      font-size: 0.75rem;
      font-weight: 500;
      cursor: pointer;
      transition: all 0.4s ease;
      background: linear-gradient(135deg, rgba(255, 255, 255, 0.9), rgba(255, 255, 255, 0.8));
      backdrop-filter: blur(20px);
      color: #2d3436;
      box-shadow: 0 6px 15px rgba(0,0,0,0.12);
      border: 1px solid rgba(255, 255, 255, 0.3);
      z-index: 1000;
    }
  
    .previous-button-responsive:hover {
      transform: translateY(-1px);
      box-shadow: 0 8px 20px rgba(0,0,0,0.18);
      background: linear-gradient(135deg, rgba(255, 255, 255, 0.95), rgba(255, 255, 255, 0.9));
    }
  }
  
  /* Tablet and below - Hide the responsive button, use navigation instead */
  @media (max-width: 1199px) {
    .previous-button-responsive {
      display: none;
    }
  }
  
  /* Rank Tag Chip - Responsive Positioning */
  .rank-tag-chip {
    position: absolute;
    top: 15px;
    left: 15px;
    padding: 6px 10px;
    border-radius: 15px;
    font-size: 0.7rem;
    font-weight: 600;
    color: white;
    text-transform: uppercase;
    letter-spacing: 0.3px;
    box-shadow: 0 3px 8px rgba(0, 0, 0, 0.15);
    z-index: 10;
    animation: tagChipSlideIn 0.4s ease-out;
  }
  
  /* Desktop/Laptop - Rank tag in top left of question card (consistent across all screens) */
  @media (min-width: 1200px) {
    .rank-tag-chip {
      left: 20px;
      top: 20px;
      padding: 6px 12px;
      font-size: 0.75rem;
    }
  }
  
  .rank-tag-chip.level-1 {
    background: linear-gradient(135deg, #00b894, #00cec9);
  }
  
  .rank-tag-chip.level-2 {
    background: linear-gradient(135deg, #fdcb6e, #f39c12);
  }
  
  .rank-tag-chip.level-3 {
    background: linear-gradient(135deg, #e17055, #d63031);
  }
  
  .rank-tag-chip.level-4 {
    background: linear-gradient(135deg, #d63031, #c0392b);
  }
  
  .rank-tag-chip.level-0 {
    background: linear-gradient(135deg, #636e72, #2d3436);
  }
  
  @keyframes tagChipSlideIn {
    0% {
      transform: translateX(20px) scale(0.8);
      opacity: 0;
    }
    100% {
      transform: translateX(0) scale(1);
      opacity: 1;
    }
  }
  
  .gentle-question {
    font-size: 1.35rem;
    color: #2d3436 !important;
    margin-bottom: 15px;
    line-height: 1.5;
    font-weight: 400;
    text-align: center;
  }
  
  .question-header {
    display: flex;
    flex-direction: column;
    align-items: center;
    margin-bottom: 10px;
  }
  
  .question-description {
    color: #636e72;
    margin-bottom: 15px;
    font-style: italic;
    text-align: center;
  }
  
  /* Dynamic Question Section */
  .question-section {
    text-align: center;
    padding: 10px;
    border-radius: 15px;
    transition: background-color 0.3s ease;
    min-height: 120px;
  }
  
  .answers-container {
    display: flex;
    flex-direction: column;
    gap: 6px;
    max-width: 400px;
    margin: 0 auto;
  }
  
  .answer-option {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 12px;
    border: 2px solid rgba(224, 224, 224, 0.6);
    border-radius: 15px;
    cursor: pointer;
    transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
    margin-bottom: 8px;
    position: relative;
    overflow: hidden;
    background: linear-gradient(135deg, rgba(255, 255, 255, 0.95), rgba(255, 255, 255, 0.9));
    backdrop-filter: blur(15px);
    box-shadow: 
      0 6px 20px rgba(0, 0, 0, 0.08),
      0 0 0 1px rgba(255, 255, 255, 0.2),
      inset 0 1px 0 rgba(255, 255, 255, 0.3);
  }
  
  .answer-option:hover {
    transform: translateY(-4px) scale(1.02);
    box-shadow: 
      0 20px 40px rgba(0, 0, 0, 0.15),
      0 0 0 1px rgba(255, 255, 255, 0.4),
      inset 0 1px 0 rgba(255, 255, 255, 0.5);
    border-color: rgba(116, 185, 255, 0.6);
    background: linear-gradient(135deg, rgba(255, 255, 255, 1), rgba(255, 255, 255, 0.98));
  }
  
  .answer-option.selected:not(.bg-red-600):not(.bg-orange-500):not(.bg-yellow-400):not(.bg-green-600) {
    border: none;
    background: linear-gradient(135deg, rgba(255, 255, 255, 0.95), rgba(255, 255, 255, 0.85));
    box-shadow: 
      0 20px 40px rgba(116, 185, 255, 0.3),
      0 8px 16px rgba(0, 0, 0, 0.1),
      inset 0 1px 0 rgba(255, 255, 255, 0.8);
    transform: translateY(-4px) scale(1.02);
    position: relative;
    overflow: hidden;
    animation: selectedPulse 2s ease-in-out infinite;
  }
  
  @keyframes selectedPulse {
    0%, 100% { 
      box-shadow: 
        0 20px 40px rgba(116, 185, 255, 0.3),
        0 8px 16px rgba(0, 0, 0, 0.1),
        inset 0 1px 0 rgba(255, 255, 255, 0.8);
    }
    50% { 
      box-shadow: 
        0 20px 40px rgba(116, 185, 255, 0.4),
        0 8px 16px rgba(0, 0, 0, 0.15),
        inset 0 1px 0 rgba(255, 255, 255, 0.9);
    }
  }
  
  /* Override for answers with CSS colors - keep their original colors when selected */
  .answer-option.bg-red-600.selected,
  .answer-option.bg-orange-500.selected,
  .answer-option.bg-yellow-400.selected,
  .answer-option.bg-green-600.selected {
    transform: translateY(-4px) scale(1.02);
    box-shadow: 0 15px 30px rgba(0, 0, 0, 0.25);
  }
  
  /* NUCLEAR OPTION: API colors NEVER change when selected - EVER! */
  .answer-option.bg-red-600.selected {
    background: #dc2626 !important;
    background-color: #dc2626 !important;
    background-image: none !important;
    color: white !important;
    border: 2px solid #dc2626 !important;
    animation: none !important;
    transform: translateY(-4px) scale(1.02) !important;
    box-shadow: 0 15px 30px rgba(220, 38, 38, 0.3) !important;
  }
  
  .answer-option.bg-orange-500.selected {
    background: #f97316 !important;
    background-color: #f97316 !important;
    background-image: none !important;
    color: black !important;
    border: 2px solid #f97316 !important;
    animation: none !important;
    transform: translateY(-4px) scale(1.02) !important;
    box-shadow: 0 15px 30px rgba(249, 115, 22, 0.3) !important;
  }
  
  .answer-option.bg-yellow-400.selected {
    background: #facc15 !important;
    background-color: #facc15 !important;
    background-image: none !important;
    color: black !important;
    border: 2px solid #facc15 !important;
    animation: none !important;
    transform: translateY(-4px) scale(1.02) !important;
    box-shadow: 0 15px 30px rgba(250, 204, 21, 0.3) !important;
  }
  
  .answer-option.bg-green-600.selected {
    background: #16a34a !important;
    background-color: #16a34a !important;
    background-image: none !important;
    color: white !important;
    border: 2px solid #16a34a !important;
    animation: none !important;
    transform: translateY(-4px) scale(1.02) !important;
    box-shadow: 0 15px 30px rgba(22, 163, 74, 0.3) !important;
  }
  
  .answer-option.selected::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    height: 3px;
    background: linear-gradient(90deg, #74b9ff, #0984e3, #00b894);
    border-radius: 15px 15px 0 0;
  }
  
  .answer-option.selected::after {
    content: '✓';
    position: absolute;
    top: 8px;
    right: 8px;
    width: 18px;
    height: 18px;
    background: linear-gradient(135deg, #74b9ff, #0984e3);
    color: white;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 10px;
    font-weight: bold;
    box-shadow: 0 2px 6px rgba(116, 185, 255, 0.4);
    animation: checkmarkPulse 0.6s ease-out;
    z-index: 10;
    cursor: pointer;
  }
  
  .answer-option.selected:hover::after {
    background: linear-gradient(135deg, #e74c3c, #c0392b);
    transform: scale(1.1);
    transition: all 0.2s ease;
  }
  
  /* Ensure API colored answers keep their checkmark styling and don't change colors */
  .answer-option.bg-red-600.selected::after,
  .answer-option.bg-orange-500.selected::after,
  .answer-option.bg-yellow-400.selected::after,
  .answer-option.bg-green-600.selected::after {
    background: linear-gradient(135deg, #74b9ff, #0984e3) !important;
    color: white !important;
  }
  
  .answer-option.bg-red-600.selected:hover::after,
  .answer-option.bg-orange-500.selected:hover::after,
  .answer-option.bg-yellow-400.selected:hover::after,
  .answer-option.bg-green-600.selected:hover::after {
    background: linear-gradient(135deg, #74b9ff, #0984e3) !important;
    transform: scale(1.1);
  }
  
  /* Rank-based answer option styling - only when no CSS is provided */
  .answer-option.rank-1:not(.bg-red-600):not(.bg-orange-500):not(.bg-yellow-400):not(.bg-green-600) {
    background: linear-gradient(135deg, rgba(0, 184, 148, 0.15), rgba(0, 206, 201, 0.1));
    border: 2px solid rgba(0, 184, 148, 0.3);
    color: #2d3436;
  }
  
  .answer-option.rank-2:not(.bg-red-600):not(.bg-orange-500):not(.bg-yellow-400):not(.bg-green-600) {
    background: linear-gradient(135deg, rgba(253, 203, 110, 0.15), rgba(243, 156, 18, 0.1));
    border: 2px solid rgba(253, 203, 110, 0.3);
    color: #2d3436;
  }
  
  .answer-option.rank-3:not(.bg-red-600):not(.bg-orange-500):not(.bg-yellow-400):not(.bg-green-600) {
    background: linear-gradient(135deg, rgba(225, 112, 85, 0.15), rgba(214, 48, 49, 0.1));
    border: 2px solid rgba(225, 112, 85, 0.3);
    color: #2d3436;
  }
  
  .answer-option.rank-4:not(.bg-red-600):not(.bg-orange-500):not(.bg-yellow-400):not(.bg-green-600) {
    background: linear-gradient(135deg, rgba(214, 48, 49, 0.15), rgba(192, 57, 43, 0.1));
    border: 2px solid rgba(214, 48, 49, 0.3);
    color: white;
  }
  
  .answer-option.rank-0:not(.bg-red-600):not(.bg-orange-500):not(.bg-yellow-400):not(.bg-green-600) {
    background: linear-gradient(135deg, rgba(99, 110, 114, 0.15), rgba(45, 52, 54, 0.1));
    border: 2px solid rgba(99, 110, 114, 0.3);
    color: #2d3436;
  }
  
  .answer-option.rank-1.selected:not(.bg-red-600):not(.bg-orange-500):not(.bg-yellow-400):not(.bg-green-600) {
    background: linear-gradient(135deg, rgba(0, 184, 148, 0.3), rgba(0, 206, 201, 0.2)) !important;
    border: 2px solid #00b894 !important;
    transform: translateY(-4px) scale(1.02);
    box-shadow: 0 15px 30px rgba(0, 184, 148, 0.3);
  }
  
  .answer-option.rank-2.selected:not(.bg-red-600):not(.bg-orange-500):not(.bg-yellow-400):not(.bg-green-600) {
    background: linear-gradient(135deg, rgba(253, 203, 110, 0.3), rgba(243, 156, 18, 0.2)) !important;
    border: 2px solid #fdcb6e !important;
    transform: translateY(-4px) scale(1.02);
    box-shadow: 0 15px 30px rgba(253, 203, 110, 0.3);
  }
  
  .answer-option.rank-3.selected:not(.bg-red-600):not(.bg-orange-500):not(.bg-yellow-400):not(.bg-green-600) {
    background: linear-gradient(135deg, rgba(225, 112, 85, 0.3), rgba(214, 48, 49, 0.2)) !important;
    border: 2px solid #e17055 !important;
    transform: translateY(-4px) scale(1.02);
    box-shadow: 0 15px 30px rgba(225, 112, 85, 0.3);
  }
  
  .answer-option.rank-4.selected:not(.bg-red-600):not(.bg-orange-500):not(.bg-yellow-400):not(.bg-green-600) {
    background: linear-gradient(135deg, rgba(214, 48, 49, 0.4), rgba(192, 57, 43, 0.3)) !important;
    border: 2px solid #d63031 !important;
    transform: translateY(-4px) scale(1.02);
    box-shadow: 0 15px 30px rgba(214, 48, 49, 0.3);
    color: white !important;
  }
  
  .answer-option.rank-0.selected:not(.bg-red-600):not(.bg-orange-500):not(.bg-yellow-400):not(.bg-green-600) {
    background: linear-gradient(135deg, rgba(99, 110, 114, 0.3), rgba(45, 52, 54, 0.2)) !important;
    border: 2px solid #636e72 !important;
    transform: translateY(-4px) scale(1.02);
    box-shadow: 0 15px 30px rgba(99, 110, 114, 0.3);
  }
  
  .answer-option.rank-1.selected::before {
    background: linear-gradient(90deg, #00b894, #00cec9);
  }
  
  .answer-option.rank-2.selected::before {
    background: linear-gradient(90deg, #fdcb6e, #f39c12);
  }
  
  .answer-option.rank-3.selected::before {
    background: linear-gradient(90deg, #e17055, #d63031);
  }
  
  .answer-option.rank-4.selected::before {
    background: linear-gradient(90deg, #d63031, #c0392b);
  }
  
  .answer-option.rank-0.selected::before {
    background: linear-gradient(90deg, #636e72, #2d3436);
  }
  
  .answer-option.rank-1.selected::after {
    background: linear-gradient(135deg, #00b894, #00cec9);
  }
  
  .answer-option.rank-2.selected::after {
    background: linear-gradient(135deg, #fdcb6e, #f39c12);
  }
  
  .answer-option.rank-3.selected::after {
    background: linear-gradient(135deg, #e17055, #d63031);
  }
  
  .answer-option.rank-4.selected::after {
    background: linear-gradient(135deg, #d63031, #c0392b);
  }
  
  .answer-option.rank-0.selected::after {
    background: linear-gradient(135deg, #636e72, #2d3436);
  }
  
  /* Default white styling for answers without CSS */
  .answer-option.default-white {
    background: rgba(255, 255, 255, 0.9);
    border-color: rgba(224, 224, 224, 0.8);
  }
  
  .answer-option.default-white:hover {
    background: rgba(255, 255, 255, 1);
    border-color: rgba(116, 185, 255, 0.6);
  }
  
  .answer-option.default-white.selected {
    background: rgba(255, 255, 255, 0.95);
    border-color: rgba(116, 185, 255, 0.8);
  }
  
  /* Tailwind-like CSS classes from API - High Priority */
  .answer-option.bg-red-600 {
    background: #dc2626 !important;
    background-color: #dc2626 !important;
    color: white !important;
    border: 2px solid #dc2626 !important;
  }
  
  .answer-option.bg-orange-500 {
    background: #f97316 !important;
    background-color: #f97316 !important;
    color: black !important;
    border: 2px solid #f97316 !important;
  }
  
  .answer-option.bg-yellow-400 {
    background: #facc15 !important;
    background-color: #facc15 !important;
    color: black !important;
    border: 2px solid #facc15 !important;
  }
  
  .answer-option.bg-green-600 {
    background: #16a34a !important;
    background-color: #16a34a !important;
    color: white !important;
    border: 2px solid #16a34a !important;
  }
  
  .answer-option.text-white {
    color: white !important;
  }
  
  .answer-option.text-black {
    color: black !important;
  }
  
  /* Override default styling when API CSS is provided */
  .answer-option.bg-red-600,
  .answer-option.bg-orange-500,
  .answer-option.bg-yellow-400,
  .answer-option.bg-green-600 {
    border-color: transparent;
  }
  
  .answer-option.bg-red-600:hover,
  .answer-option.bg-orange-500:hover,
  .answer-option.bg-yellow-400:hover,
  .answer-option.bg-green-600:hover {
    transform: translateY(-3px);
    box-shadow: 0 15px 30px rgba(0, 0, 0, 0.2);
  }
  
  .answer-option.bg-red-600.selected,
  .answer-option.bg-orange-500.selected,
  .answer-option.bg-yellow-400.selected,
  .answer-option.bg-green-600.selected {
    transform: translateY(-4px) scale(1.02);
    box-shadow: 0 20px 40px rgba(0, 0, 0, 0.25);
  }
  
  .answer-content {
    flex: 1;
    text-align: left;
    position: relative;
    z-index: 1;
  }
  
  .answer-text {
    font-weight: 500;
    color: #1a202c;
    margin-bottom: 4px;
    font-size: 0.9rem;
    line-height: 1.3;
  }
  
  .answer-rank {
    font-size: 0.85rem;
    color: #6b7280;
    font-weight: 600;
    background: rgba(116, 185, 255, 0.1);
    padding: 3px 6px;
    border-radius: 8px;
    display: inline-block;
  }
  
  /* Dynamic rank styling based on answer rank */
  .answer-option.rank-1 .answer-rank {
    background: rgba(0, 184, 148, 0.1);
    color: #00b894;
  }
  
  .answer-option.rank-2 .answer-rank {
    background: rgba(253, 203, 110, 0.1);
    color: #f39c12;
  }
  
  .answer-option.rank-3 .answer-rank {
    background: rgba(225, 112, 85, 0.1);
    color: #e17055;
  }
  
  .answer-option.rank-4 .answer-rank {
    background: rgba(214, 48, 49, 0.1);
    color: #d63031;
  }
  
  .answer-option.rank-0 .answer-rank {
    background: rgba(99, 110, 114, 0.1);
    color: #636e72;
  }
  
  @keyframes checkmarkPulse {
    0% {
      transform: scale(0);
      opacity: 0;
    }
    50% {
      transform: scale(1.2);
    }
    100% {
      transform: scale(1);
      opacity: 1;
    }
  }
  
  /* Navigation */
  .supportive-navigation {
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 25px 35px;
    background: rgba(255, 255, 255, 0.1);
    backdrop-filter: blur(10px);
    border-radius: 0 0 30px 30px;
    margin: 0 20px;
    gap: 20px;
  }
  
  .encouragement-container {
    text-align: center;
    padding: 15px 20px;
    margin: 0 20px;
  }
  
  .nav-cloud {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 6px 12px;
    border: none;
    border-radius: 18px;
    font-size: 0.8rem;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.4s ease;
    background: linear-gradient(135deg, #ffffff, #f8f9ff);
    color: #2d3436;
    box-shadow: 0 5px 15px rgba(0,0,0,0.1);
  }
  
  .nav-cloud:hover {
    transform: translateY(-2px);
    box-shadow: 0 10px 25px rgba(0,0,0,0.15);
  }
  
  .nav-cloud:disabled {
    opacity: 0.5;
    cursor: not-allowed;
    transform: none;
  }
  
  .next-cloud {
    background: linear-gradient(135deg, #74b9ff, #0984e3);
    color: white;
  }
  
  
  
  .encouragement-message {
    color: rgba(255, 255, 255, 0.9);
    font-size: 0.9rem;
    font-style: italic;
  }
  
  /* Results */
  .healing-results {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0,0,0,0.8);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
    backdrop-filter: blur(5px);
    overflow-y: auto;
    padding: 20px 0;
  }
  
  .results-sanctuary {
    background: linear-gradient(135deg, #ffffff, #f8f9ff);
    padding: 40px;
    border-radius: 30px;
    text-align: center;
    max-width: 600px !important;
    width: 90% !important;
    max-height: 90vh;
    box-shadow: 
      0 30px 80px rgba(0,0,0,0.3),
      0 0 0 1px rgba(255, 255, 255, 0.2),
      inset 0 1px 0 rgba(255, 255, 255, 0.4);
    position: relative;
    overflow: hidden;
    margin: auto;
    overflow-y: auto;
    backdrop-filter: blur(20px);
    border: 1px solid rgba(255, 255, 255, 0.3);
  }
  
  .results-sanctuary::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    height: 5px;
    background: linear-gradient(90deg, #00b894, #00cec9, #74b9ff, #a29bfe);
  }
  
  /* Celebration and Results Styles */
  .celebration-header {
    margin-bottom: 30px;
  }
  
  .celebration-icon {
    font-size: 4rem;
    margin-bottom: 15px;
    animation: gentleBounce 2s ease-in-out infinite;
  }
  
  .celebration-title {
    font-size: 2.2rem;
    color: #2d3436;
    margin-bottom: 10px;
    font-weight: 600;
  }
  
  .celebration-message {
    font-size: 1.1rem;
    color: #636e72;
    margin-bottom: 20px;
  }
  
  .results-garden {
    margin: 30px 0;
  }
  
  .level-badge {
    display: inline-flex;
    align-items: center;
    gap: 15px;
    padding: 20px 30px;
    border-radius: 20px;
    background: rgba(255, 255, 255, 0.9);
    backdrop-filter: blur(10px);
    border: 1px solid rgba(255, 255, 255, 0.2);
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
    transition: all 0.3s ease;
  }
  
  .level-badge.green {
    background: linear-gradient(135deg, rgba(0, 184, 148, 0.1), rgba(0, 206, 201, 0.1));
    border-color: rgba(0, 184, 148, 0.3);
  }
  
  .level-badge.yellow {
    background: linear-gradient(135deg, rgba(253, 203, 110, 0.1), rgba(243, 156, 18, 0.1));
    border-color: rgba(253, 203, 110, 0.3);
  }
  
  .level-badge.orange {
    background: linear-gradient(135deg, rgba(225, 112, 85, 0.1), rgba(214, 48, 49, 0.1));
    border-color: rgba(225, 112, 85, 0.3);
  }
  
  .level-badge.red {
    background: linear-gradient(135deg, rgba(214, 48, 49, 0.1), rgba(192, 57, 43, 0.1));
    border-color: rgba(214, 48, 49, 0.3);
  }
  
  .badge-icon {
    font-size: 2.5rem;
  }
  
  .badge-text {
    font-size: 1.1rem;
    font-weight: 600;
    color: #2d3436;
  }
  
  .healing-message {
    margin: 30px 0;
    padding: 25px;
    background: linear-gradient(135deg, rgba(116, 185, 255, 0.1), rgba(9, 132, 227, 0.1));
    border-radius: 15px;
    border: 1px solid rgba(116, 185, 255, 0.2);
    display: flex;
    align-items: center;
    gap: 15px;
  }
  
  .message-icon {
    font-size: 2rem;
    color: #74b9ff;
  }
  
  
  
  .support-actions {
    display: flex;
    gap: 20px;
    justify-content: center;
    margin: 25px 0;
    flex-wrap: wrap;
  }
  
  .support-button {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 15px 25px;
    border: none;
    border-radius: 25px;
    font-size: 1rem;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.4s ease;
  }
  
  .restart-button {
    background: linear-gradient(135deg, #74b9ff, #0984e3);
    color: white;
  }
  
  .submit-button {
    background: linear-gradient(135deg, #00b894, #00cec9);
    color: white;
  }
  
  
  
  .support-button:hover {
    transform: translateY(-2px);
    box-shadow: 0 10px 25px rgba(0,0,0,0.2);
  }
  
  .support-button:disabled {
    opacity: 0.6;
    cursor: not-allowed;
    transform: none;
  }
  
  .support-button:disabled:hover {
    transform: none;
    box-shadow: 0 5px 15px rgba(0,0,0,0.2);
  }
  
  .final-encouragement {
    margin-top: 25px;
    padding: 20px;
    background: linear-gradient(135deg, #a8e6cf, #dcedc1);
    border-radius: 15px;
    color: #2d3436;
    font-style: italic;
    line-height: 1.6;
  }
  
  /* Landing Page Content */
  .landing-page-content {
    text-align: left;
    line-height: 1.8;
    color: #2d3436;
    background: linear-gradient(135deg, rgba(255, 255, 255, 0.1), rgba(255, 255, 255, 0.05));
    padding: 25px;
    border-radius: 20px;
    border: 1px solid rgba(255, 255, 255, 0.2);
    margin: 20px 0;
    backdrop-filter: blur(10px);
    box-shadow: 
      0 10px 25px rgba(0, 0, 0, 0.1),
      inset 0 1px 0 rgba(255, 255, 255, 0.2);
  }
  
  .landing-page-content h2 {
    font-size: 1.8rem;
    margin-bottom: 20px;
    font-weight: 600;
    background: linear-gradient(135deg, #b00020, #d32f2f);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
    text-shadow: 0 2px 4px rgba(176, 0, 32, 0.1);
  }
  
  .landing-page-content p {
    font-size: 1.1rem;
    margin-bottom: 18px;
    color: #2d3436;
    font-weight: 500;
  }
  
  .landing-page-content ul {
    margin: 20px 0;
    padding-left: 0;
    list-style: none;
  }
  
  .landing-page-content li {
    margin-bottom: 12px;
    font-size: 1rem;
    padding: 12px 20px;
    background: linear-gradient(135deg, rgba(176, 0, 32, 0.05), rgba(211, 47, 47, 0.03));
    border-radius: 12px;
    border-left: 4px solid #b00020;
    position: relative;
    font-weight: 500;
    color: #2d3436;
    box-shadow: 0 4px 12px rgba(176, 0, 32, 0.1);
  }
  
  .landing-page-content li::before {
    content: '•';
    color: #b00020;
    font-weight: bold;
    position: absolute;
    left: 8px;
    top: 50%;
    transform: translateY(-50%);
    font-size: 1.2rem;
  }
  
  
  
  
  /* Special styling for stoplight survey */
  .stoplight-survey .gentle-question {
    font-size: 2rem;
    font-weight: 700;
    margin-bottom: 15px;
  }
  
  .stoplight-survey .gentle-question .desc {
    font-size: 1rem;
    font-weight: 400;
    color: #636e72;
    font-style: italic;
  }
  
  .stoplight-survey .question-description {
    display: none;
  }
  
  .stoplight-survey .answer-option {
    margin-bottom: 10px;
    padding: 15px 20px;
  }
  
  .stoplight-survey .answer-text {
    font-size: 1rem;
    font-weight: 500;
  }
  
  .stoplight-survey .answer-rank {
    display: none;
  }
  
  /* Dynamic Continue button styling */
  .continue-button-after-selected {
    display: flex;
    justify-content: flex-end;
    margin: 10px 0;
    padding-right: 15px;
    animation: slideInUp 0.3s ease-out;
  }
  
  .continue-button-dynamic {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 8px 16px;
    border: none;
    border-radius: 20px;
    font-size: 0.85rem;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.4s ease;
    background: linear-gradient(135deg, #74b9ff, #0984e3);
    color: white;
    box-shadow: 0 4px 12px rgba(0,0,0,0.15);
  }
  
  .continue-button-dynamic:hover {
    transform: translateY(-1px);
    box-shadow: 0 6px 20px rgba(0,0,0,0.25);
  }
  
  @keyframes slideInUp {
    from {
      opacity: 0;
      transform: translateY(10px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }
  
  
  
  /* Enhanced Responsive Design */
  
  /* Desktop First Approach - Base styles are for desktop */
  
  /* Large Desktop (1200px and up) - Perfect 2 Column Layout */
  @media (min-width: 1200px) {
    .therapeutic-container {
      width: 100%;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      gap: 20px;
      padding: 30px 40px;
      max-width: 1700px;
      margin: 0 auto;
      min-height: 100vh;
      overflow: hidden;
      position: relative;
    }
    
    .therapeutic-container::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      background: 
        radial-gradient(circle at 20% 20%, rgba(116, 185, 255, 0.1) 0%, transparent 50%),
        radial-gradient(circle at 80% 80%, rgba(245, 87, 108, 0.1) 0%, transparent 50%),
        radial-gradient(circle at 40% 60%, rgba(240, 147, 251, 0.1) 0%, transparent 50%);
      pointer-events: none;
      z-index: 0;
    }
    
    .survey-config-container {
      width: 100%;
      max-width: 450px;
      margin: 0;
      padding: 25px;
      display: block; /* Show dropdowns on laptop screens */
      background: linear-gradient(135deg, rgba(255, 255, 255, 0.25), rgba(255, 255, 255, 0.15));
      backdrop-filter: blur(20px);
      border-radius: 25px;
      border: 1px solid rgba(255, 255, 255, 0.4);
      box-shadow: 
        0 20px 40px rgba(0, 0, 0, 0.15),
        0 0 0 1px rgba(255, 255, 255, 0.2),
        inset 0 1px 0 rgba(255, 255, 255, 0.3);
      transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
      position: relative;
      z-index: 2;
      align-self: flex-start;
    }
    
    .survey-config-container:hover {
      transform: translateY(-3px);
      box-shadow: 
        0 25px 50px rgba(0, 0, 0, 0.2),
        0 0 0 1px rgba(255, 255, 255, 0.3),
        inset 0 1px 0 rgba(255, 255, 255, 0.4);
    }
    
    .survey-config-container-mobile {
      display: none; /* Hide mobile dropdowns on laptop screens */
    }
    
    .config-dropdowns {
      justify-content: center;
      max-width: 100%;
      margin: 0;
      gap: 15px;
      flex-wrap: wrap;
    }
    
    .config-dropdown {
      background: linear-gradient(135deg, rgba(255, 255, 255, 0.95), rgba(255, 255, 255, 0.9));
      border: 2px solid rgba(255, 255, 255, 0.5);
      border-radius: 15px;
      padding: 14px 16px;
      font-size: 0.9rem;
      font-weight: 500;
      color: #2d3436;
      backdrop-filter: blur(20px);
      transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
      cursor: pointer;
      outline: none;
      box-shadow: 
        0 8px 25px rgba(0, 0, 0, 0.1),
        0 0 0 1px rgba(255, 255, 255, 0.2),
        inset 0 1px 0 rgba(255, 255, 255, 0.6);
      min-width: 180px;
      flex: 1;
      appearance: none;
      -webkit-appearance: none;
      -moz-appearance: none;
      background-image: url("data:image/svg+xml;charset=UTF-8,%3csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='%232d3436' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3e%3cpolyline points='6,9 12,15 18,9'%3e%3c/polyline%3e%3c/svg%3e");
      background-repeat: no-repeat !important;
      background-position: right 12px center !important;
      background-size: 16px 16px !important;
      padding-right: 40px;
    }
    
    .config-dropdown:hover {
      background: linear-gradient(135deg, rgba(255, 255, 255, 1), rgba(255, 255, 255, 0.98));
      border-color: rgba(116, 185, 255, 0.7);
      transform: translateY(-1px);
      box-shadow: 
        0 12px 30px rgba(0, 0, 0, 0.12),
        0 0 0 1px rgba(116, 185, 255, 0.3),
        inset 0 1px 0 rgba(255, 255, 255, 0.7);
      background-image: url("data:image/svg+xml;charset=UTF-8,%3csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='%2374b9ff' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3e%3cpolyline points='6,9 12,15 18,9'%3e%3c/polyline%3e%3c/svg%3e");
      background-repeat: no-repeat !important;
      background-position: right 12px center !important;
      background-size: 16px 16px !important;
    }
    
    .config-dropdown:focus {
      border-color: rgba(116, 185, 255, 0.8);
      box-shadow: 
        0 12px 30px rgba(116, 185, 255, 0.15),
        0 0 0 3px rgba(116, 185, 255, 0.15),
        inset 0 1px 0 rgba(255, 255, 255, 0.7);
      transform: translateY(-1px);
      background-image: url("data:image/svg+xml;charset=UTF-8,%3csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='%2374b9ff' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3e%3cpolyline points='6,9 12,15 18,9'%3e%3c/polyline%3e%3c/svg%3e");
      background-repeat: no-repeat !important;
      background-position: right 12px center !important;
      background-size: 16px 16px !important;
    }
    
    .main-content-wrapper {
      display: flex;
      flex-direction: row;
      align-items: flex-start;
      justify-content: center;
      gap: 50px;
      width: 100%;
      order: 2;
      padding: 0 20px;
      position: relative;
      z-index: 1;
      margin-top: 0;
    }
    
    .left-column {
      display: flex;
      flex-direction: column;
      align-items: center;
      flex: 0 0 450px;
      gap: 25px;
      position: relative;
      align-self: flex-start;
    }
    
    .right-column {
      display: flex;
      flex-direction: column;
      align-items: center;
      flex: 0 0 auto;
      align-self: flex-start;
    }
    
    .gentle-header {
      max-width: 100%;
      padding: 45px 40px;
      flex: 0 0 auto;
      background: linear-gradient(135deg, rgba(255, 255, 255, 0.25), rgba(255, 255, 255, 0.15));
      backdrop-filter: blur(25px);
      border-radius: 35px;
      border: 2px solid rgba(255, 255, 255, 0.5);
      box-shadow: 
        0 30px 60px rgba(0, 0, 0, 0.2),
        0 0 0 1px rgba(255, 255, 255, 0.2),
        inset 0 2px 0 rgba(255, 255, 255, 0.4);
      text-align: center;
      transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
      align-self: center;
      position: relative;
      overflow: hidden;
    }
    
    .gentle-header::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      height: 3px;
      background: linear-gradient(90deg, #667eea 0%, #764ba2 25%, #f093fb 50%, #f5576c 75%, #4facfe 100%);
      opacity: 0.9;
      border-radius: 35px 35px 0 0;
    }
    
    .gentle-header:hover {
      transform: translateY(-5px) scale(1.02);
      box-shadow: 
        0 40px 80px rgba(0, 0, 0, 0.25),
        0 0 0 1px rgba(255, 255, 255, 0.3),
        inset 0 2px 0 rgba(255, 255, 255, 0.5);
    }
    
    /* Enhanced progress section inside left column */
    .progress-section {
      position: relative;
      margin-top: 25px;
      background: linear-gradient(135deg, rgba(255, 255, 255, 0.2), rgba(255, 255, 255, 0.1));
      backdrop-filter: blur(25px);
      border-radius: 25px;
      padding: 30px;
      border: 1px solid rgba(255, 255, 255, 0.4);
      width: 100%;
      box-sizing: border-box;
      box-shadow: 
        0 20px 40px rgba(0, 0, 0, 0.15),
        0 0 0 1px rgba(255, 255, 255, 0.2),
        inset 0 2px 0 rgba(255, 255, 255, 0.3);
      transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
      overflow: hidden;
    }
    
    .progress-section::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      height: 2px;
      background: linear-gradient(90deg, #667eea 0%, #764ba2 25%, #f093fb 50%, #f5576c 75%, #4facfe 100%);
      opacity: 0.8;
    }
    
    .progress-section:hover {
      transform: translateY(-3px);
      box-shadow: 
        0 25px 50px rgba(0, 0, 0, 0.2),
        0 0 0 1px rgba(255, 255, 255, 0.3),
        inset 0 2px 0 rgba(255, 255, 255, 0.4);
    }
    
    /* Circular Progress Bar for Laptop */
    .circular-progress-container {
      display: flex;
      justify-content: center;
      align-items: center;
    }
    
    .circular-progress-wrapper {
      position: relative;
      display: flex;
      align-items: center;
      justify-content: center;
      animation: gentleFloat 6s ease-in-out infinite;
    }
    
    .circular-progress {
      transform: rotate(-90deg);
      filter: drop-shadow(0 0 10px rgba(116, 185, 255, 0.3));
    }
    
    .circular-progress-wrapper:hover {
      animation-play-state: paused;
      transform: scale(1.05);
    }
    
    .progress-bg {
      stroke: rgba(255, 255, 255, 0.2);
      stroke-linecap: round;
    }
    
    .progress-circle {
      stroke-linecap: round;
      transition: stroke-dashoffset 0.8s cubic-bezier(0.4, 0, 0.2, 1);
      filter: drop-shadow(0 0 8px rgba(116, 185, 255, 0.4));
    }
    
    .circular-progress-content {
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      text-align: center;
      color: white;
    }
    
    .progress-percentage {
      font-size: 1.8rem;
      font-weight: 600;
      text-shadow: 0 2px 10px rgba(0,0,0,0.3);
      margin-bottom: 2px;
      animation: progressPulse 2s ease-in-out infinite;
    }
    
    .progress-step {
      font-size: 0.9rem;
      font-weight: 500;
      opacity: 0.9;
      text-shadow: 0 1px 5px rgba(0,0,0,0.2);
    }
    
    @keyframes progressPulse {
      0%, 100% { 
        transform: scale(1);
        text-shadow: 0 2px 10px rgba(0,0,0,0.3);
      }
      50% { 
        transform: scale(1.05);
        text-shadow: 
          0 2px 10px rgba(0,0,0,0.3),
          0 0 20px rgba(116, 185, 255, 0.4);
      }
    }
    
    /* Linear Progress Bar for Mobile/Tablet */
    .linear-progress-container {
      display: none;
    }
    
    
    .question-sanctuary {
      max-width: 100%;
      min-width: 500px;
      margin: 0;
      flex: 0 0 auto;
      align-self: center;
      margin-top: 0;
    }
    
    .question-cloud {
      background: linear-gradient(135deg, rgba(255, 255, 255, 0.95), rgba(255, 255, 255, 0.9));
      backdrop-filter: blur(30px);
      border-radius: 30px;
      padding: 35px;
      box-shadow: 
        0 30px 60px rgba(0, 0, 0, 0.2),
        0 0 0 1px rgba(255, 255, 255, 0.4),
        inset 0 2px 0 rgba(255, 255, 255, 0.5);
      border: 2px solid rgba(255, 255, 255, 0.6);
      transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
      position: relative;
      overflow: hidden;
    }
    
    .question-cloud::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      height: 3px;
      background: linear-gradient(90deg, #667eea 0%, #764ba2 25%, #f093fb 50%, #f5576c 75%, #4facfe 100%);
      opacity: 0.8;
      border-radius: 30px 30px 0 0;
    }
    
    .question-cloud:hover {
      transform: translateY(-5px) scale(1.01);
      box-shadow: 
        0 40px 80px rgba(0, 0, 0, 0.25),
        0 0 0 1px rgba(255, 255, 255, 0.5),
        inset 0 2px 0 rgba(255, 255, 255, 0.6);
    }
    
    .results-sanctuary {
      max-width: 100%;
    }
  }
  
  /* Desktop (992px to 1199px) - Single Column */
  @media (max-width: 1199px) and (min-width: 992px) {
    .therapeutic-container {
      padding: 0 20px;
      display: block;
    }
    
    .gentle-header {
      max-width: 600px;
      margin: 0 auto;
    }
    
    .progress-section {
      max-width: 600px;
      margin-left: auto;
      margin-right: auto;
    }
    
    .question-sanctuary {
      max-width: 600px;
      margin: 0 auto 25px;
    }
    
    .results-sanctuary {
      max-width: 650px;
    }
  }
  
  /* Tablet Landscape (768px to 991px) - Compact Layout */
  @media (max-width: 991px) and (min-width: 768px) {
    .therapeutic-container {
      padding: 0 10px;
    }
    
    .gentle-header {
      padding: 10px 15px;
      max-width: 100%;
    }
    
    .breath-circle {
      width: 25px;
      height: 25px;
    }
    
    .caring-title {
      font-size: 1.8rem;
    }
    
    .progress-section {
      padding: 10px;
      margin: 3px 0;
      max-width: 100%;
      width: calc(100% - 20px);
      margin-left: 10px;
      margin-right: 10px;
    }
    
    .question-sanctuary {
      max-width: 100%;
      margin: 0 auto 10px;
    }
    
    .question-cloud {
      padding: 20px 18px;
      min-height: 320px;
    }
    
    .gentle-question {
      font-size: 1.1rem;
    }
    
    .answers-container {
      max-width: 100%;
      gap: 8px;
    }
    
    .answer-option {
      padding: 12px 15px;
      margin-bottom: 8px;
    }
    
    .answer-text {
      font-size: 0.85rem;
      line-height: 1.2;
    }
    
    .supportive-navigation {
      padding: 20px 25px;
      margin: 0 15px;
    }
    
    .support-actions {
      flex-direction: row;
      justify-content: center;
      gap: 15px;
      flex-wrap: wrap;
    }
    
    .results-sanctuary {
      padding: 35px 25px;
      margin: 15px;
      max-width: 100%;
    }
    
    .celebration-title {
      font-size: 2rem;
    }
    
    .error-title {
      font-size: 1.8rem;
    }
    
    .debug-button {
      bottom: 15px;
      right: 15px;
    }
  }
  
  /* Tablet Portrait (576px to 767px) - Balanced Spacing */
  @media (max-width: 767px) and (min-width: 576px) {
    .therapeutic-container {
      padding: 0 12px;
      min-height: 100vh;
    }
    
    .floating-elements {
      display: none; /* Hide floating elements on smaller screens for better performance */
    }
    
    .gentle-header {
      padding: 12px 16px;
      max-width: 100%;
    }
    
    .breathing-icon {
      margin-bottom: 8px;
    }
    
    .breath-circle {
      width: 35px;
      height: 35px;
    }
    
    .caring-title {
      font-size: 1.6rem;
      margin-bottom: 6px;
    }
    
    .supportive-message {
      font-size: 0.95rem;
      margin-bottom: 4px;
    }
    
    .encouragement {
      font-size: 0.85rem;
      margin-bottom: 12px;
    }
    
    .progress-section {
      padding: 12px 16px;
      margin: 8px 0;
      border-radius: 12px;
      max-width: 100%;
      width: calc(100% - 24px);
      margin-left: 12px;
      margin-right: 12px;
    }
    
    .progress-bar-header {
      flex-direction: row;
      gap: 8px;
      align-items: center;
      margin-bottom: 8px;
    }
    
    .progress-text {
      font-size: 0.9rem;
    }
    
    .step-counter {
      font-size: 0.8rem;
      padding: 3px 6px;
    }
    
    .progress-bar-track {
      height: 6px;
    }
    
    .question-sanctuary {
      max-width: 100%;
      margin: 0 auto 15px;
    }
    
    .progress-border-container {
      margin: 8px;
    }
    
    .question-cloud {
      padding: 16px 20px;
      min-height: 320px;
      border-radius: 18px;
    }
    
    .rank-tag-chip {
      top: 12px;
      left: 12px;
      padding: 5px 10px;
      font-size: 0.75rem;
    }
    
    .question-section {
      padding: 12px 8px;
      min-height: 240px;
    }
    
    .question-header {
      margin-bottom: 12px;
    }
    
    .gentle-question {
      font-size: 1.2rem;
      margin-bottom: 8px;
    }
    
    .question-description {
      font-size: 0.9rem;
      margin-bottom: 15px;
    }
    
    .answers-container {
      max-width: 100%;
      gap: 8px;
    }
    
    .answer-option {
      padding: 12px 15px;
      margin-bottom: 6px;
      border-radius: 10px;
      min-height: 50px;
    }
    
    .answer-text {
      font-size: 0.85rem;
      margin-bottom: 4px;
      line-height: 1.3;
    }
    
    .answer-rank {
      font-size: 0.75rem;
      padding: 3px 6px;
    }
    
    .continue-button-after-selected {
      margin: 12px 0;
      padding-right: 12px;
    }
    
    .continue-button-dynamic {
      padding: 10px 16px;
      font-size: 0.9rem;
    }
    
    .supportive-navigation {
      flex-direction: row;
      gap: 12px;
      padding: 12px 16px;
      margin: 0 12px;
      border-radius: 0 0 18px 18px;
    }
    
    .nav-cloud {
      padding: 10px 16px;
      font-size: 0.9rem;
    }
    
    .encouragement-container {
      padding: 12px 16px;
      margin: 0 12px;
    }
    
    .encouragement-message {
      font-size: 0.9rem;
    }
    
    .support-actions {
      flex-direction: row;
      align-items: center;
      gap: 8px;
      margin: 8px 0;
      justify-content: center;
    }
    
    .support-button {
      padding: 6px 12px;
      font-size: 0.8rem;
      min-width: 120px;
    }
    
    .results-sanctuary {
      padding: 15px 12px;
      margin: 5px;
      max-width: 100%;
      border-radius: 15px;
      max-height: 95vh;
      overflow-y: auto;
    }
    
    .celebration-header {
      margin-bottom: 15px;
    }
    
    .celebration-icon {
      font-size: 2.5rem;
      margin-bottom: 8px;
    }
    
    .celebration-title {
      font-size: 1.4rem;
      margin-bottom: 4px;
    }
    
    .celebration-message {
      font-size: 0.9rem;
      margin-bottom: 12px;
    }
    
    .level-badge {
      padding: 12px 15px;
      gap: 8px;
    }
    
    .badge-icon {
      font-size: 1.8rem;
    }
    
    .badge-text {
      font-size: 0.9rem;
    }
    
    .healing-message {
      margin: 15px 0;
      padding: 12px;
      gap: 8px;
    }
    
    .message-icon {
      font-size: 1.4rem;
    }
    
    .submission-status {
      margin: 12px 0;
      padding: 12px;
    }
    
    .submission-loading p,
    .submission-success p,
    .submission-pending p {
      font-size: 0.8rem;
    }
    
    .final-encouragement {
      margin-top: 12px;
      padding: 12px;
      font-size: 0.8rem;
    }
    
    .landing-page-content h2 {
      font-size: 1.2rem;
      margin-bottom: 8px;
    }
    
    .landing-page-content p {
      font-size: 0.9rem;
      margin-bottom: 8px;
    }
    
    .landing-page-content li {
      font-size: 0.8rem;
      margin-bottom: 4px;
    }
    
    .error-container {
      padding: 12px;
    }
    
    .error-icon {
      font-size: 2.5rem;
      margin-bottom: 10px;
    }
    
    .error-title {
      font-size: 1.3rem;
      margin-bottom: 8px;
    }
    
    .error-message {
      font-size: 0.9rem;
      margin-bottom: 15px;
    }
    
    .retry-button {
      padding: 8px 12px;
      font-size: 0.8rem;
    }
    
    .debug-button {
      bottom: 8px;
      right: 8px;
    }
    
    .debug-modal-header {
      padding: 4px 8px;
    }
    
    .debug-modal-header h3 {
      font-size: 0.8rem;
    }
    
    .debug-modal-body {
      padding: 8px;
    }
    
    .debug-json {
      padding: 8px;
      font-size: 10px;
    }
  }
  
  /* Mobile Landscape (480px to 575px) - Balanced Spacing */
  @media (max-width: 575px) and (min-width: 480px) {
    .therapeutic-container {
      padding: 0 8px;
      min-height: 100vh;
    }
    
    .gentle-header {
      padding: 8px 12px;
    }
    
    .breath-circle {
      width: 30px;
      height: 30px;
    }
    
    .caring-title {
      font-size: 1.4rem;
      margin-bottom: 5px;
    }
    
    .supportive-message {
      font-size: 0.9rem;
      margin-bottom: 3px;
    }
    
    .encouragement {
      font-size: 0.8rem;
      margin-bottom: 10px;
    }
    
    .progress-section {
      padding: 10px 12px;
      margin: 6px 0;
      max-width: 100%;
      width: calc(100% - 16px);
      margin-left: 8px;
      margin-right: 8px;
    }
    
    .progress-bar-header {
      margin-bottom: 6px;
    }
    
    .progress-text {
      font-size: 0.8rem;
    }
    
    .step-counter {
      font-size: 0.75rem;
      padding: 2px 5px;
    }
    
    .progress-bar-track {
      height: 5px;
    }
    
    .question-sanctuary {
      margin: 0 auto 12px;
      flex: 1;
      display: flex;
      flex-direction: column;
    }
    
    .progress-border-container {
      margin: 6px;
      flex: 1;
      display: flex;
      flex-direction: column;
    }
    
    .question-cloud {
      padding: 12px 15px;
      min-height: 280px;
      border-radius: 12px;
      flex: 1;
      display: flex;
      flex-direction: column;
    }
    
    .rank-tag-chip {
      top: 10px;
      left: 10px;
      padding: 4px 8px;
      font-size: 0.7rem;
    }
    
    .question-section {
      flex: 1;
      display: flex;
      flex-direction: column;
      padding: 10px 6px;
      min-height: 200px;
    }
    
    .question-header {
      margin-bottom: 10px;
    }
    
    .gentle-question {
      font-size: 1.1rem;
      margin-bottom: 6px;
    }
    
    .question-description {
      font-size: 0.8rem;
      margin-bottom: 12px;
    }
    
    .answers-container {
      gap: 6px;
      flex: 1;
      display: flex;
      flex-direction: column;
      justify-content: space-between;
    }
    
    .answer-option {
      padding: 10px 12px;
      margin-bottom: 5px;
      border-radius: 8px;
      flex: 1;
      min-height: 45px;
    }
    
    .answer-text {
      font-size: 0.75rem;
      margin-bottom: 3px;
      line-height: 1.2;
    }
    
    .answer-rank {
      font-size: 0.7rem;
      padding: 2px 5px;
    }
    
    .continue-button-after-selected {
      margin: 8px 0;
      padding-right: 8px;
    }
    
    .continue-button-dynamic {
      padding: 6px 12px;
      font-size: 0.8rem;
    }
    
    .supportive-navigation {
      flex-direction: row;
      gap: 8px;
      padding: 8px 12px;
      margin: 0 8px;
      border-radius: 0 0 12px 12px;
    }
    
    .nav-cloud {
      padding: 6px 12px;
      font-size: 0.8rem;
    }
    
    .encouragement-container {
      padding: 8px 12px;
      margin: 0 8px;
    }
    
    .encouragement-message {
      font-size: 0.8rem;
    }
    
    .support-actions {
      flex-direction: row;
      align-items: center;
      gap: 5px;
      margin: 5px 0;
      justify-content: center;
    }
    
    .support-button {
      padding: 4px 8px;
      font-size: 0.7rem;
      min-width: 100px;
    }
    
    .results-sanctuary {
      padding: 10px 8px;
      margin: 3px;
      border-radius: 10px;
      max-height: 95vh;
      overflow-y: auto;
    }
    
    .celebration-header {
      margin-bottom: 10px;
    }
    
    .celebration-icon {
      font-size: 2rem;
      margin-bottom: 5px;
    }
    
    .celebration-title {
      font-size: 1.2rem;
      margin-bottom: 3px;
    }
    
    .celebration-message {
      font-size: 0.8rem;
      margin-bottom: 8px;
    }
    
    .level-badge {
      padding: 8px 10px;
      gap: 5px;
    }
    
    .badge-icon {
      font-size: 1.5rem;
    }
    
    .badge-text {
      font-size: 0.8rem;
    }
    
    .healing-message {
      margin: 10px 0;
      padding: 8px;
      gap: 5px;
    }
    
    .message-icon {
      font-size: 1.2rem;
    }
    
    .submission-status {
      margin: 8px 0;
      padding: 8px;
    }
    
    .submission-loading p,
    .submission-success p,
    .submission-pending p {
      font-size: 0.7rem;
    }
    
    .final-encouragement {
      margin-top: 8px;
      padding: 8px;
      font-size: 0.7rem;
    }
    
    .landing-page-content h2 {
      font-size: 1.1rem;
      margin-bottom: 5px;
    }
    
    .landing-page-content p {
      font-size: 0.8rem;
      margin-bottom: 5px;
    }
    
    .landing-page-content li {
      font-size: 0.7rem;
      margin-bottom: 3px;
    }
    
    .error-container {
      padding: 8px;
    }
    
    .error-icon {
      font-size: 2rem;
      margin-bottom: 8px;
    }
    
    .error-title {
      font-size: 1.1rem;
      margin-bottom: 5px;
    }
    
    .error-message {
      font-size: 0.8rem;
      margin-bottom: 10px;
    }
    
    .retry-button {
      padding: 5px 8px;
      font-size: 0.7rem;
    }
  }
  
  /* Mobile Portrait (up to 479px) - Balanced Spacing */
  @media (max-width: 479px) {
    .therapeutic-container {
      padding: 0 6px;
      min-height: 100vh;
    }
    
    .gentle-header {
      padding: 6px 10px;
    }
    
    .breath-circle {
      width: 25px;
      height: 25px;
    }
    
    .caring-title {
      font-size: 1.2rem;
      margin-bottom: 4px;
    }
    
    .supportive-message {
      font-size: 0.85rem;
      margin-bottom: 2px;
    }
    
    .encouragement {
      font-size: 0.75rem;
      margin-bottom: 8px;
    }
    
    .progress-section {
      padding: 8px 10px;
      margin: 4px 0;
      border-radius: 10px;
      max-width: 100%;
      width: calc(100% - 12px);
      margin-left: 6px;
      margin-right: 6px;
    }
    
    .progress-bar-header {
      margin-bottom: 4px;
    }
    
    .progress-text {
      font-size: 0.75rem;
    }
    
    .step-counter {
      font-size: 0.7rem;
      padding: 2px 4px;
    }
    
    .progress-bar-track {
      height: 5px;
    }
    
    .question-sanctuary {
      margin: 0 auto 10px;
      flex: 1;
      display: flex;
      flex-direction: column;
    }
    
    .progress-border-container {
      margin: 4px;
      flex: 1;
      display: flex;
      flex-direction: column;
    }
    
    .question-cloud {
      padding: 10px 12px;
      min-height: 250px;
      border-radius: 10px;
      flex: 1;
      display: flex;
      flex-direction: column;
    }
    
    .rank-tag-chip {
      top: 8px;
      left: 8px;
      padding: 3px 6px;
      font-size: 0.65rem;
    }
    
    .question-section {
      flex: 1;
      display: flex;
      flex-direction: column;
      padding: 8px 4px;
      min-height: 160px;
    }
    
    .question-header {
      margin-bottom: 8px;
    }
    
    .gentle-question {
      font-size: 1rem;
      margin-bottom: 4px;
    }
    
    .question-description {
      font-size: 0.75rem;
      margin-bottom: 10px;
    }
    
    .answers-container {
      gap: 4px;
      flex: 1;
      display: flex;
      flex-direction: column;
      justify-content: space-between;
    }
    
    .answer-option {
      padding: 8px 10px;
      margin-bottom: 4px;
      border-radius: 6px;
      flex: 1;
      min-height: 40px;
    }
    
    .answer-text {
      font-size: 0.7rem;
      margin-bottom: 2px;
      line-height: 1.2;
    }
    
    .answer-rank {
      font-size: 0.65rem;
      padding: 2px 4px;
    }
    
    .continue-button-after-selected {
      margin: 6px 0;
      padding-right: 6px;
    }
    
    .continue-button-dynamic {
      padding: 5px 10px;
      font-size: 0.75rem;
    }
    
    .supportive-navigation {
      flex-direction: row;
      gap: 6px;
      padding: 6px 10px;
      margin: 0 6px;
      border-radius: 0 0 10px 10px;
    }
    
    .nav-cloud {
      padding: 5px 10px;
      font-size: 0.75rem;
    }
    
    .encouragement-container {
      padding: 6px 10px;
      margin: 0 6px;
    }
    
    .encouragement-message {
      font-size: 0.75rem;
    }
    
    .support-actions {
      flex-direction: row;
      align-items: center;
      gap: 3px;
      margin: 3px 0;
      justify-content: center;
    }
    
    .support-button {
      padding: 3px 6px;
      font-size: 0.6rem;
      min-width: 80px;
    }
    
    .results-sanctuary {
      padding: 8px 6px;
      margin: 2px;
      border-radius: 8px;
      max-height: 95vh;
      overflow-y: auto;
    }
    
    .celebration-header {
      margin-bottom: 8px;
    }
    
    .celebration-icon {
      font-size: 1.5rem;
      margin-bottom: 3px;
    }
    
    .celebration-title {
      font-size: 1rem;
      margin-bottom: 2px;
    }
    
    .celebration-message {
      font-size: 0.7rem;
      margin-bottom: 5px;
    }
    
    .level-badge {
      padding: 5px 8px;
      gap: 3px;
    }
    
    .badge-icon {
      font-size: 1.2rem;
    }
    
    .badge-text {
      font-size: 0.7rem;
    }
    
    .healing-message {
      margin: 8px 0;
      padding: 5px;
      gap: 3px;
    }
    
    .message-icon {
      font-size: 1rem;
    }
    
    .submission-status {
      margin: 5px 0;
      padding: 5px;
    }
    
    .submission-loading p,
    .submission-success p,
    .submission-pending p {
      font-size: 0.6rem;
    }
    
    .final-encouragement {
      margin-top: 5px;
      padding: 5px;
      font-size: 0.6rem;
    }
    
    .landing-page-content h2 {
      font-size: 0.9rem;
      margin-bottom: 3px;
    }
    
    .landing-page-content p {
      font-size: 0.7rem;
      margin-bottom: 3px;
    }
    
    .landing-page-content li {
      font-size: 0.6rem;
      margin-bottom: 2px;
    }
    
    .error-container {
      padding: 5px;
    }
    
    .error-icon {
      font-size: 1.5rem;
      margin-bottom: 5px;
    }
    
    .error-title {
      font-size: 0.9rem;
      margin-bottom: 3px;
    }
    
    .error-message {
      font-size: 0.7rem;
      margin-bottom: 8px;
    }
    
    .retry-button {
      padding: 3px 6px;
      font-size: 0.6rem;
    }
    
    .debug-button {
      bottom: 5px;
      right: 5px;
    }
    
    .code-icon-circle {
      width: 20px;
      height: 20px;
    }
    
    .code-brackets {
      font-size: 10px;
    }
    
    .debug-modal-header {
      padding: 3px 6px;
    }
    
    .debug-modal-header h3 {
      font-size: 0.7rem;
    }
    
    .debug-modal-body {
      padding: 6px;
    }
    
    .debug-json {
      padding: 6px;
      font-size: 8px;
    }
  }
  
  /* Touch-friendly improvements for mobile devices */
  @media (hover: none) and (pointer: coarse) {
    .answer-option {
      min-height: 48px; /* Minimum touch target size */
    }
    
    .nav-cloud,
    .support-button,
    .continue-button-dynamic,
    .retry-button {
      min-height: 44px; /* Minimum touch target size */
    }
    
    .debug-button {
      min-width: 44px;
      min-height: 44px;
    }
    
    /* Remove hover effects on touch devices */
    .answer-option:hover {
      transform: none;
      box-shadow: none;
    }
    
    .nav-cloud:hover,
    .support-button:hover,
    .continue-button-dynamic:hover,
    .retry-button:hover {
      transform: none;
      box-shadow: none;
    }
    
    .debug-button:hover {
      transform: none;
    }
  }
  
  /* High DPI displays */
  @media (-webkit-min-device-pixel-ratio: 2), (min-resolution: 192dpi) {
    .breath-circle,
    .floating-circle {
      border-width: 1px; /* Thinner borders on high DPI */
    }
    
    .progress-bar-track {
      border-width: 1px;
    }
  }
  
  /* Landscape orientation adjustments */
  @media (orientation: landscape) and (max-height: 500px) {
    .therapeutic-container {
      min-height: 100vh;
    }
    
    .gentle-header {
      padding: 10px 15px;
    }
    
    .caring-title {
      font-size: 1.5rem;
      margin-bottom: 8px;
    }
    
    .supportive-message {
      font-size: 0.9rem;
      margin-bottom: 4px;
    }
    
    .encouragement {
      font-size: 0.8rem;
      margin-bottom: 10px;
    }
    
    .progress-section {
      padding: 10px 15px;
      margin: 8px 0;
      max-width: 100%;
      width: calc(100% - 16px);
      margin-left: 8px;
      margin-right: 8px;
    }
    
    .question-cloud {
      padding: 15px 20px;
      min-height: 250px;
    }
    
    .gentle-question {
      font-size: 1rem;
      margin-bottom: 15px;
    }
    
    .question-description {
      font-size: 0.8rem;
      margin-bottom: 15px;
    }
    
    .answer-option {
      padding: 10px;
      margin-bottom: 6px;
    }
    
    .answer-text {
      font-size: 0.8rem;
    }
    
    .answer-rank {
      font-size: 0.7rem;
    }
    
    .results-sanctuary {
      padding: 20px 15px;
      max-height: 90vh;
      overflow-y: auto;
    }
    
    .celebration-icon {
      font-size: 2.5rem;
      margin-bottom: 8px;
    }
    
    .celebration-title {
      font-size: 1.4rem;
      margin-bottom: 6px;
    }
    
    .celebration-message {
      font-size: 0.9rem;
      margin-bottom: 12px;
    }
    
    .level-badge {
      padding: 12px 15px;
    }
    
    .badge-icon {
      font-size: 1.8rem;
    }
    
    .badge-text {
      font-size: 0.9rem;
    }
  }
  
  /* Debug Button and Modal */
  .debug-button-container {
    position: fixed;
    bottom: 20px;
    right: 20px;
    z-index: 1000;
  }
  
  .debug-button {
    position: fixed;
    bottom: 20px;
    right: 20px;
    cursor: pointer;
    z-index: 1000;
    transition: transform 0.3s ease;
  }
  
  .debug-button:hover {
    transform: scale(1.1);
  }
  
  .debug-modal {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background: rgba(0, 0, 0, 0.8);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 9999;
    padding: 0;
  }
  
  .debug-modal-content {
    width: 100%;
    height: 100%;
    background: white;
    border-radius: 0;
    box-shadow: none;
    display: flex;
    flex-direction: column;
    max-width: none;
    max-height: none;
  }
  
  .debug-modal-header {
    background: #2c3e50;
    color: white;
    padding: 8px 20px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    border-bottom: 1px solid #34495e;
    flex-shrink: 0;
  }
  
  .debug-modal-header h3 {
    margin: 0;
    font-size: 1rem;
    font-weight: 500;
  }
  
  .debug-header-controls {
    display: flex;
    gap: 10px;
  }
  
  .debug-copy-btn {
    background: none;
    border: none;
    color: white;
    font-size: 12px;
    font-weight: 500;
    cursor: pointer;
    padding: 4px 8px;
    border-radius: 4px;
    transition: background 0.3s ease;
    text-transform: uppercase;
    letter-spacing: 0.5px;
  }
  
  .code-icon-circle {
    width: 32px;
    height: 32px;
    border-radius: 50%;
    background: linear-gradient(135deg, #667eea, #764ba2);
    display: flex;
    align-items: center;
    justify-content: center;
    box-shadow: 0 4px 12px rgba(102, 126, 234, 0.3);
    transition: all 0.3s ease;
  }
  
  .code-brackets {
    font-size: 14px;
    color: white;
    font-weight: bold;
    font-family: monospace;
  }
  
  .debug-copy-btn:hover .code-icon-circle {
    transform: scale(1.1);
    box-shadow: 0 6px 16px rgba(102, 126, 234, 0.4);
  }
  
  .debug-close-btn {
    background: none;
    border: none;
    color: white;
    font-size: 20px;
    cursor: pointer;
    padding: 0;
    width: 24px;
    height: 24px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
    transition: background 0.3s ease;
  }
  
  .debug-close-btn:hover {
    background: rgba(255, 255, 255, 0.2);
  }
  
  .debug-modal-body {
    padding: 20px;
    flex: 1;
    overflow-y: auto;
    background: #f8f9fa;
  }
  
  .debug-json {
    background: white;
    border: 1px solid #e9ecef;
    border-radius: 8px;
    padding: 20px;
    font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
    font-size: 12px;
    line-height: 1.5;
    color: #333;
    white-space: pre-wrap;
    word-wrap: break-word;
    margin: 0;
    height: 100%;
    overflow-y: auto;
  }
  
  /* Submission Status Styles */
  .submission-status {
    margin: 25px 0;
    padding: 20px;
    border-radius: 15px;
    background: rgba(255, 255, 255, 0.9);
    backdrop-filter: blur(10px);
    border: 1px solid rgba(255, 255, 255, 0.2);
  }
  
  .submission-loading,
  .submission-success,
  .submission-pending {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 15px;
    text-align: center;
  }
  
  .submission-loading p,
  .submission-success p,
  .submission-pending p {
    margin: 0;
    font-size: 1rem;
    font-weight: 500;
    color: #2d3436;
  }
  
  .loading-spinner {
    width: 20px;
    height: 20px;
    border: 2px solid #e9ecef;
    border-top: 2px solid #74b9ff;
    border-radius: 50%;
    animation: spin 1s linear infinite;
  }
  
  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }
  
  .success-icon {
    font-size: 1.5rem;
    animation: successPulse 0.6s ease-out;
  }
  
  @keyframes successPulse {
    0% {
      transform: scale(0);
      opacity: 0;
    }
    50% {
      transform: scale(1.2);
    }
    100% {
      transform: scale(1);
      opacity: 1;
    }
  }
  
  .pending-icon {
    font-size: 1.5rem;
    animation: gentleBounce 2s ease-in-out infinite;
  }
  
  @keyframes gentleBounce {
    0%, 100% { transform: translateY(0px); }
    50% { transform: translateY(-5px); }
  }
  
  
  /* Survey Configuration Dropdowns */
  .survey-config-container {
    max-width: 100%;
    margin: 0 0 15px 0;
    padding: 0;
    position: relative;
    z-index: 10;
    display: block; /* Show by default */
  }
  
  /* Mobile dropdown container - hidden by default, only shown on mobile */
  .survey-config-container-mobile {
    max-width: 100%;
    margin: 0 0 15px 0;
    padding: 0;
    position: relative;
    z-index: 10;
    display: none; /* Hidden by default */
  }
  
  .config-dropdowns {
    display: flex;
    gap: 15px;
    justify-content: center;
    flex-wrap: nowrap;
    flex-direction: row;
  }
  
  /* Basic dropdown styles - will be overridden by media queries */
  .config-dropdown {
    flex: 1;
    max-width: 50%;
    background: rgba(255, 255, 255, 0.9);
    border: 2px solid rgba(45, 52, 54, 0.2);
    border-radius: 10px;
    padding: 8px 12px;
    font-size: 0.85rem;
    color: #2d3436;
    backdrop-filter: blur(10px);
    transition: all 0.3s ease;
    cursor: pointer;
    outline: none;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    appearance: none;
    -webkit-appearance: none;
    -moz-appearance: none;
    background-image: url("data:image/svg+xml;charset=UTF-8,%3csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='%232d3436' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3e%3cpolyline points='6,9 12,15 18,9'%3e%3c/polyline%3e%3c/svg%3e");
    background-repeat: no-repeat !important;
    background-position: right 12px center !important;
    background-size: 16px 16px !important;
    padding-right: 35px;
  }
  
  .config-dropdown:hover {
    background: rgba(255, 255, 255, 1);
    border-color: rgba(45, 52, 54, 0.4);
    transform: translateY(-1px);
    box-shadow: 0 4px 15px rgba(0, 0, 0, 0.15);
    background-image: url("data:image/svg+xml;charset=UTF-8,%3csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='%232d3436' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3e%3cpolyline points='6,9 12,15 18,9'%3e%3c/polyline%3e%3c/svg%3e");
    background-repeat: no-repeat !important;
    background-position: right 12px center !important;
    background-size: 16px 16px !important;
  }
  
  .config-dropdown:focus {
    border-color: rgba(45, 52, 54, 0.6);
    box-shadow: 0 0 0 3px rgba(45, 52, 54, 0.1);
    background-image: url("data:image/svg+xml;charset=UTF-8,%3csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='%232d3436' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3e%3cpolyline points='6,9 12,15 18,9'%3e%3c/polyline%3e%3c/svg%3e");
    background-repeat: no-repeat !important;
    background-position: right 12px center !important;
    background-size: 16px 16px !important;
  }
  
  .config-dropdown option {
    background: #ffffff;
    color: #2d3436;
    padding: 12px 16px;
    border: none;
    font-weight: 500;
  }
  
  .config-dropdown option:hover {
    background: #f8f9fa;
    color: #2d3436;
  }
  
  .config-dropdown option:checked,
  .config-dropdown option:selected {
    background: #74b9ff;
    color: white;
    font-weight: 600;
  }
  
  /* Required field styling */
  .config-dropdown.required-field {
    border-color: #e74c3c !important;
    background: linear-gradient(135deg, rgba(231, 76, 60, 0.05), rgba(255, 255, 255, 0.95)) !important;
    box-shadow: 
      0 8px 25px rgba(231, 76, 60, 0.15),
      0 0 0 1px rgba(231, 76, 60, 0.3),
      inset 0 1px 0 rgba(255, 255, 255, 0.6) !important;
  }
  
  .config-dropdown.required-field:hover {
    border-color: #c0392b !important;
    background: linear-gradient(135deg, rgba(192, 57, 43, 0.08), rgba(255, 255, 255, 1)) !important;
    box-shadow: 
      0 12px 30px rgba(192, 57, 43, 0.2),
      0 0 0 1px rgba(192, 57, 43, 0.4),
      inset 0 1px 0 rgba(255, 255, 255, 0.7) !important;
  }
  
  .config-dropdown.required-field:focus {
    border-color: #a93226 !important;
    box-shadow: 
      0 12px 30px rgba(169, 50, 38, 0.2),
      0 0 0 3px rgba(169, 50, 38, 0.15),
      inset 0 1px 0 rgba(255, 255, 255, 0.7) !important;
  }
  
  /* Mobile First - Ensure row layout on all mobile devices */
  @media (max-width: 1199px) {
    .survey-config-container {
      display: none; /* Hide main dropdowns on smaller screens */
    }
    
    .survey-config-container-mobile {
      display: block; /* Show mobile dropdowns on smaller screens */
      padding: 0 15px;
      margin-bottom: 15px;
    }
    
    /* Show linear progress on smaller screens */
    .circular-progress-container {
      display: none;
    }
    
    .linear-progress-container {
      display: block;
    }
    
    .config-dropdowns {
      display: flex !important;
      flex-direction: row !important;
      gap: 8px;
      align-items: center;
      justify-content: space-between;
      flex-wrap: nowrap !important;
    }
    
    .config-dropdown {
      flex: 1 !important;
      max-width: 48% !important;
      padding: 6px 8px;
      font-size: 0.75rem;
    }
  }
  
  @media (max-width: 575px) {
    .survey-config-container-mobile {
      padding: 0 10px;
      margin-bottom: 12px;
    }
    
    .config-dropdowns {
      display: flex !important;
      flex-direction: row !important;
      gap: 6px;
      justify-content: space-between;
      flex-wrap: nowrap !important;
    }
    
    .config-dropdown {
      flex: 1 !important;
      max-width: 48% !important;
      padding: 5px 6px;
      font-size: 0.7rem;
    }
  }
  
  @media (max-width: 479px) {
    .survey-config-container-mobile {
      padding: 0 8px;
      margin-bottom: 10px;
    }
    
    .config-dropdowns {
      display: flex !important;
      flex-direction: row !important;
      gap: 4px;
      justify-content: space-between;
      flex-wrap: nowrap !important;
    }
    
    .dropdown-group {
      width: 48% !important;
      max-width: 48% !important;
      flex: 1 !important;
      min-width: 0 !important;
    }
    
    .dropdown-label {
      font-size: 0.65rem;
      text-align: center;
    }
    
    .config-dropdown {
      padding: 4px 5px;
      font-size: 0.65rem;
      width: 100%;
    }
  }
  </style>
  