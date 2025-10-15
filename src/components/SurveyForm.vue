<template>
  <div class="therapeutic-container" :class="{ 'stoplight-survey': isStoplightSurvey }">
      <!-- Previous Button - Desktop/Laptop Only -->
      <button 
        v-if="!loading && !showResults && questions.length > 0 && currentStep > 0" 
        @click="previousStep" 
        class="previous-button-responsive"
        title="Go to previous question"
      >
        <span class="button-icon">←</span>
        <span>Previous</span>
      </button>
      
      <!-- Previous Button - Mobile/Tablet Only -->
      <button 
        v-if="!loading && !showResults && questions.length > 0 && currentStep > 0" 
        @click="previousStep" 
        class="previous-button-mobile-fixed"
        title="Go to previous question"
      >
        <span class="button-icon">←</span>
        <span>Previous</span>
      </button>
      
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

        <!-- Enhanced Question Card for Laptop/Desktop -->
        <div class="enhanced-question-card">
          <!-- Survey Header with Progress -->
          <div class="survey-header">
            <div class="header-content">
              <!-- SFA Logo Section -->
              <div class="sfa-logo-section">
                <img src="https://demo.galaxycreations.com/sfa/SFA-Logo_sm.png" alt="SFA Logo" class="sfa-logo" />
              </div>
              
              <div class="survey-title-section">
                <!-- Title Row with Rank Tag and Title -->
                <div class="title-row">
                  <!-- Rank Tag Chip -->
                  <div v-if="currentQuestion" class="rank-tag-chip" :class="getQuestionLevelClass()">
                    {{ getRankDisplayText() }}
                  </div>
                  <h1 class="survey-title">{{ surveyData.surveyName || 'Patient Assessment' }}</h1>
                </div>
                <p class="survey-subtitle">Take a moment to check in with yourself. Your feelings matter. 💙</p>
              </div>
              
              <!-- NJ Logo Section -->
              <div class="nj-logo-section">
                <img src="https://demo.galaxycreations.com/sfa/NJnew-solid-Black-Tag_sm.png" alt="NJ Logo" class="nj-logo" />
              </div>
              
              
            </div>
          </div>

          <!-- Survey Configuration Dropdowns -->
          <div class="survey-config-section">
            <div class="config-dropdowns">
              <!-- Custom Dropdown that can be auto-opened -->
              <div class="custom-dropdown-container">
                <div 
                  @click="toggleShiftDropdown" 
                  class="config-dropdown custom-dropdown" 
                  :class="{ 'required-field': !selectedShift, 'dropdown-open': showShiftDropdown }"
                  :disabled="loadingShifts"
                >
                  <span>{{ getSelectedShiftText() || 'Select Shift *' }}</span>
                  <svg class="dropdown-arrow" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
                    <path d="M7 10L12 15L17 10H7Z" fill="currentColor"/>
                  </svg>
                </div>
                
                <div v-if="showShiftDropdown" class="dropdown-options">
                  <div 
                    v-for="shift in shiftOptions" 
                    :key="shift.lookupID"
                    @click="selectShift(shift.lookupID, shift.descr)"
                    class="dropdown-option"
                  >
                    {{ shift.descr }}
                  </div>
                </div>
              </div>
              
              <!-- Custom Station Dropdown -->
              <div class="custom-dropdown-container">
                <div 
                  @click="toggleStationDropdown" 
                  class="config-dropdown custom-dropdown" 
                  :class="{ 'required-field': !selectedLocation, 'dropdown-open': showStationDropdown }"
                  :disabled="loadingDevices"
                >
                  <span>{{ getSelectedStationText() || 'Select Stations *' }}</span>
                  <svg class="dropdown-arrow" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
                    <path d="M7 10L12 15L17 10H7Z" fill="currentColor"/>
                  </svg>
                </div>
                
                <div v-if="showStationDropdown" class="dropdown-options">
                  <div 
                    v-for="device in stationDevices" 
                    :key="device.stationID"
                    @click="selectStation(device.codeID, device.customLabel1)"
                    class="dropdown-option"
                  >
                    {{ device.customLabel1 }}
                  </div>
                </div>
              </div>
            </div>
          </div>

          <!-- Main Content Area -->
          <div class="question-sanctuary">
            <!-- Progress Border Container -->
            <div class="progress-border-container">
              <!-- Question Content -->
              <div 
                class="question-cloud"
                :style="getQuestionCloudStyle()"
              >
                <!-- Dynamic Question Display -->
                <div v-if="currentQuestion" class="question-section">
                  <div class="question-header">
                    <h2 class="gentle-question" v-html="currentQuestion.text"></h2>
                  </div>
                  <p class="question-description">Please select the option that best describes your current situation</p>
                  
                  <div class="answers-container" :class="{ 'disabled-container': !selectedLocation }">
                    <!-- Device selection message overlay -->
                    <div v-if="!selectedLocation" class="device-selection-message">
                      <div class="message-text">
                        Please select a device first to answer the survey questions
                      </div>
                    </div>
                    
                    <template v-for="answer in currentQuestion.answers">
                      <div
                        :key="answer.answerID"
                        class="answer-option"
                        :class="{ 
                          'selected': isAnswerSelected(currentQuestion.questionID, answer.answerID),
                          'checkbox-mode': currentQuestion.questionType === 2,
                          'disabled': !selectedLocation,
                          [`rank-${answer.rank}`]: true,
                          ...getAnswerCSSClasses(answer)
                        }"
                        :style="getAnswerOptionStyle(answer)"
                        @click="selectAnswer(currentQuestion.questionID, answer.answerID)"
                      >
                        <div class="answer-content">
                          <!-- Checkbox for questionType 2 -->
                          <div v-if="currentQuestion.questionType === 2" class="checkbox-container">
                            <input 
                              type="checkbox" 
                              :checked="isAnswerSelected(currentQuestion.questionID, answer.answerID)"
                              class="answer-checkbox"
                            />
                          </div>
                          <div class="answer-text" v-html="answer.text"></div>
                        </div>
                      </div>
                      
                      <!-- Continue button for single select (questionType 1) -->
                      <div 
                        v-if="currentQuestion.questionType === 1 && selectedAnswers[currentQuestion.questionID] === answer.answerID && canProceed" 
                        :key="`continue-${answer.answerID}`"
                        class="continue-button-after-selected"
                      >
                        <button @click="nextStep" class="continue-button-dynamic">
                          <span>{{ getNextButtonText() }}</span>
                          <span class="nav-icon">🌟</span>
                        </button>
                      </div>
                    </template>
                    
                    <!-- Continue button for multiple select (questionType 2) -->
                    <div 
                      v-if="currentQuestion.questionType === 2 && canProceed" 
                      class="continue-button-after-selected"
                    >
                      <button @click="nextStep" class="continue-button-dynamic">
                        <span>{{ getNextButtonText() }}</span>
                        <span class="nav-icon">🌟</span>
                      </button>
                    </div>
                    
                  </div>
                </div>
                
              </div>
            </div>
          </div>
        </div>

      
          
        </div>
      <!-- </div> -->

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
            <p class="celebration-message">Thank you, your survey is submitted</p>
          </div>
        </div>
        
        <!-- Take Assessment Again Button - Always visible on results page -->
        <div class="results-top-actions">
          <button 
            @click="refreshPage" 
            class="support-button restart-button"
          >
            <span class="button-icon">🔄</span>
            <span>Restart Survey</span>
          </button>
        </div>

        <!-- Timer - Show immediately after survey completion -->
        <div class="submission-status">
          <div class="circular-timer-container">
            <div class="circular-timer">
              <div class="timer-content">
                <div class="timer-number">{{ countdownTimer }}</div>
                <div class="timer-label">seconds</div>
              </div>
            </div>
            <!-- Timer Message (Hide when countdown reaches 0) -->
            <div v-if="countdownTimer > 0" class="timer-message">
              <p>Page will auto-reload in {{ countdownTimer }} seconds</p>
            </div>
          </div>
        </div>
        
        
          <div class="final-encouragement">
            <p>Remember: Your wellbeing matters. Take care of yourself. 💙</p>
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
      initialized: false, // Add this to track if component has been initialized
      submittingSurvey: false, // Track survey submission status
      surveySubmitted: false, // Track if survey has been submitted
      selectedShift: '', // Track selected shift
      selectedShiftText: '', // Track selected shift text
      selectedLocation: '', // Track selected location (now device name)
      selectedStationText: '', // Track selected station text
      stationDevices: [], // Store station devices from API
      showShiftDropdown: false, // Control custom dropdown visibility
      showStationDropdown: false, // Control station dropdown visibility
      loadingDevices: false, // Track loading state for devices
      shiftOptions: [], // Store shift options from API
      loadingShifts: false, // Track loading state for shifts
      pendingAutoAdvance: false, // Track if we need to auto-advance after shift selection
      selectedDevice: null, // Store the complete selected device object
      countdownTimer: 20, // Countdown timer starting from 20 seconds
      timerInterval: null, // Store the timer interval reference
      // showFlashMessage: false, // Control flash message visibility
      flashMessageTimer: null, // Store flash message timer reference
      flashMessageType: 'success' // 'success' or 'error'
    }
  },
  
  mounted() {
    // Fetch station devices
    this.fetchStationDevices();
    // Fetch shift options from API
    this.fetchShiftOptions();
  },
  
  
  beforeDestroy() {
    // Clean up timers when component is destroyed
    this.stopTimer();
  },
  computed: {
    // Check if this is a conditional questionnaire
    isComplexQuestionnaire() {
      return this.edges.length > 0 || 
             this.landingPages.length > 0 || 
             this.surveyData.defaultLandingPageID !== undefined;
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
      
      // Don't show continue button if autoRollUp is 1 (auto-advance)
      if (this.currentQuestion.autoRollUp === 1) {
        return false;
      }
      
      if (this.currentQuestion.questionType === 2) {
        // Multiple select - check if at least one answer is selected
        return this.selectedAnswers[this.currentQuestion.questionID] && 
               this.selectedAnswers[this.currentQuestion.questionID].length > 0;
      } else {
        // Single select - check if an answer is selected
        return this.selectedAnswers[this.currentQuestion.questionID] !== undefined;
      }
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
    },
    
  },
  
  methods: {
    // Timer methods for countdown functionality
    startTimer() {
      this.countdownTimer = 20; // Reset to 20 seconds
      
      // Show success flash message immediately
      this.flashMessageType = 'success';
      // this.showFlashMessage = true;
      
      // Hide flash message after 5 seconds
      // this.flashMessageTimer = setTimeout(() => {
      //   this.showFlashMessage = false;
      // }, 5000);
      
      this.timerInterval = setInterval(() => {
        this.countdownTimer--;
        if (this.countdownTimer <= 0) {
          this.stopTimer();
          // Auto-reload the page after 20 seconds
          console.log('Timer completed - reloading page...');
          window.location.reload();
        }
      }, 1000);
    },
    
    // Show error flash message
    // showErrorFlash() {
    //   this.flashMessageType = 'error';
    //   this.showFlashMessage = true;
      
    //   // Hide flash message after 5 seconds
    //   this.flashMessageTimer = setTimeout(() => {
    //     this.showFlashMessage = false;
    //   }, 5000);
    // },
    
    stopTimer() {
      if (this.timerInterval) {
        clearInterval(this.timerInterval);
        this.timerInterval = null;
      }
      if (this.flashMessageTimer) {
        clearTimeout(this.flashMessageTimer);
        this.flashMessageTimer = null;
      }
    },
    
    resetTimer() {
      this.stopTimer();
      this.countdownTimer = 20;
      // this.showFlashMessage = false;
    },
    

    // Refresh the page
    refreshPage() {
      window.location.reload();
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
        
        const response = await fetch('https://demo.galaxycreations.com/gwas/api/ReportDefaults/rptStationDevices/80');
        
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const devices = await response.json();
        
        // Filter out devices with empty customLabel1 or codeID
        this.stationDevices = devices.filter(device => 
          device.customLabel1 && 
          device.customLabel1.trim() !== '' &&
          device.codeID &&
          device.codeID.trim() !== ''
        );
        
        console.log('Station devices loaded:', this.stationDevices);
        
        // Auto-select station based on sID query parameter
        this.autoSelectStationFromURL();
        
        
      } catch (error) {
        console.error('Error fetching station devices:', error);
        
       
        
        // Auto-select station based on sID query parameter (fallback case)
        this.autoSelectStationFromURL();
        
        // Show error message to user
        this.$nextTick(() => {
          alert('Unable to load device list. Using default options.');
        });
      } finally {
        this.loadingDevices = false;
      }
    },

    // Fetch shift options from API
    async fetchShiftOptions() {
      try {
        this.loadingShifts = true;
        console.log('Fetching shift options...');
        
        const response = await fetch('https://demo.galaxycreations.com/gwas/api/Lookups/9');
        
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const shifts = await response.json();
        this.shiftOptions = shifts;
        
        console.log('Shift options loaded:', this.shiftOptions);
        
        // Auto-open shift dropdown after data is loaded
        this.autoOpenShiftDropdown();

      } catch (error) {
        console.error('Error fetching shift options:', error);
        this.shiftOptions = [];
      } finally {
        this.loadingShifts = false;
      }
    },
    
    // Auto-open shift dropdown when component loads
    autoOpenShiftDropdown() {
      // Wait for DOM to be ready and dropdowns to be loaded
      this.$nextTick(() => {
        setTimeout(() => {
          if (!this.selectedShift && this.shiftOptions.length > 0) {
            // Open custom dropdown
            this.showShiftDropdown = true;
            console.log('Shift dropdown auto-opened');
          }
        }, 1000); // 1 second delay to ensure everything is loaded
      });
    },
    
    // Toggle custom dropdown
    toggleShiftDropdown() {
      this.showShiftDropdown = !this.showShiftDropdown;
    },
    
    // Select shift option
    selectShift(shiftId, shiftText) {
      this.selectedShift = shiftId;
      this.selectedShiftText = shiftText;
      this.showShiftDropdown = false; // Close dropdown after selection
      this.onShiftChange(); // Call the original change handler
      
      // Auto-open stations dropdown after shift selection
      setTimeout(() => {
        if (!this.selectedLocation && this.stationDevices.length > 0) {
          this.showStationDropdown = true;
          console.log('Stations dropdown auto-opened after shift selection');
        }
      }, 300); // Small delay for smooth transition
    },
    
    // Get selected shift text for display
    getSelectedShiftText() {
      return this.selectedShiftText;
    },
    
    // Toggle station dropdown
    toggleStationDropdown() {
      this.showStationDropdown = !this.showStationDropdown;
    },
    
    // Select station option
    selectStation(deviceCode, deviceText) {
      this.selectedLocation = deviceCode;
      this.selectedStationText = deviceText;
      this.showStationDropdown = false;
      
      // Find the selected device object by codeID
      this.selectedDevice = this.stationDevices.find(device => 
        device.codeID === deviceCode
      );
      
      console.log('Selected station:', deviceText, 'Code:', deviceCode);
      console.log('Selected device:', this.selectedDevice);
    },
    
    // Get selected station text for display
    getSelectedStationText() {
      return this.selectedStationText;
    },
    
    // Handle device selection from dropdown
    onDeviceSelection() {
      if (this.selectedLocation) {
        // Find the selected device object by codeID
        this.selectedDevice = this.stationDevices.find(device => 
          device.codeID === this.selectedLocation
        );
        console.log('Selected device:', this.selectedDevice);
      } else {
        this.selectedDevice = null;
      }
    },
    
    // Get survey ID from URL parameters
    getSurveyIdFromURL() {
      const urlParams = new URLSearchParams(window.location.search);
      const surveyIdParam = urlParams.get('surveyId') || urlParams.get('surveyID') || urlParams.get('id');
      
      if (surveyIdParam) {
        this.surveyId = parseInt(surveyIdParam);
        console.log('Survey ID from URL:', this.surveyId);
      } else {
        console.log('No survey ID in URL, using default:', this.surveyId);
      }
    },
    
    // Get station ID from URL parameters
    getStationIdFromURL() {
      const urlParams = new URLSearchParams(window.location.search);
      const stationIdParam = urlParams.get('sID');
      
      if (stationIdParam) {
        console.log('Station ID from URL:', stationIdParam);
        return stationIdParam;
      }
      return null;
    },
    
    // Auto-select station based on sID query parameter
    autoSelectStationFromURL() {
      const stationIdFromURL = this.getStationIdFromURL();
      
      if (stationIdFromURL && this.stationDevices.length > 0) {
        // Find the device with matching codeID
        const matchingDevice = this.stationDevices.find(device => 
          device.codeID === stationIdFromURL
        );
        
        if (matchingDevice) {
          // Auto-select the matching station
          this.selectedLocation = matchingDevice.codeID;
          this.selectedStationText = matchingDevice.customLabel1;
          this.selectedDevice = matchingDevice;
          console.log('Auto-selected station:', matchingDevice.customLabel1, 'with codeID:', matchingDevice.codeID);
        } else {
          console.log('No matching station found for sID:', stationIdFromURL);
          // Reset to default state when no matching station found
          this.selectedLocation = '';
          this.selectedStationText = '';
          this.selectedDevice = null;
        }
      } else {
        // No sID parameter present, ensure default state
        this.selectedLocation = '';
        this.selectedStationText = '';
        this.selectedDevice = null;
        console.log('No sID parameter found, showing default station selection');
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
        
        // Log autoRollUp and questionType information for debugging
        console.log('Survey questions with autoRollUp and questionType status:');
        this.questions.forEach(q => {
          console.log(`Question ${q.questionID} (${q.text}): autoRollUp = ${q.autoRollUp}, questionType = ${q.questionType} (${q.questionType === 1 ? 'Single Select' : 'Multiple Select'})`);
        });
        
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
    
    // Check if an answer is selected (works for both single and multiple select)
    isAnswerSelected(questionId, answerId) {
      const question = this.questions.find(q => q.questionID === questionId);
      if (!question) return false;
      
      if (question.questionType === 2) {
        // Multiple select - check if answerId is in the array
        return this.selectedAnswers[questionId] && this.selectedAnswers[questionId].includes(answerId);
      } else {
        // Single select - check if answerId matches
        return this.selectedAnswers[questionId] === answerId;
      }
    },

    selectAnswer(questionId, answerId) {
      // Check if required fields are filled before allowing answer selection
      if (!this.selectedLocation) {
        alert('Please select a device/location first before answering the survey questions.');
        return;
      }
      
      const question = this.questions.find(q => q.questionID === questionId);
      if (!question) return;
      
      if (question.questionType === 2) {
        // Multiple select (checkbox) behavior
        if (!this.selectedAnswers[questionId]) {
          this.$set(this.selectedAnswers, questionId, []);
        }
        
        const selectedAnswers = this.selectedAnswers[questionId];
        const index = selectedAnswers.indexOf(answerId);
        
        if (index > -1) {
          // Answer is already selected, remove it
          selectedAnswers.splice(index, 1);
          console.log('Deselected answer for question:', questionId, answerId);
        } else {
          // Answer is not selected, add it
          selectedAnswers.push(answerId);
          console.log('Selected answer for question:', questionId, answerId);
        }
        
        // Update the reactive array
        this.$set(this.selectedAnswers, questionId, [...selectedAnswers]);
      } else {
        // Single select behavior (existing logic)
        if (this.selectedAnswers[questionId] === answerId) {
          this.$delete(this.selectedAnswers, questionId);
          console.log('Deselected answer for question:', questionId);
        } else {
          this.$set(this.selectedAnswers, questionId, answerId);
          console.log('Selected answer for question:', questionId, answerId);
        }
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
      
      // Check for autoRollUp functionality - auto-advance if autoRollUp is 1
      const currentQuestion = this.questions.find(q => q.questionID === questionId);
      if (currentQuestion && currentQuestion.autoRollUp === 1) {
        // For single select, auto-advance immediately after selection
        // For multiple select, auto-advance after any selection (user can select multiple but we advance after first)
        if (currentQuestion.questionType === 1 && this.selectedAnswers[questionId]) {
          console.log('AutoRollUp detected for single select question:', questionId, '- checking if can auto-advance');
          this.handleAutoRollUpAdvance();
        } else if (currentQuestion.questionType === 2 && this.selectedAnswers[questionId] && this.selectedAnswers[questionId].length > 0) {
          console.log('AutoRollUp detected for multiple select question:', questionId, '- checking if can auto-advance');
          this.handleAutoRollUpAdvance();
        }
      }
    },

    // Handle autoRollUp advance with shift validation
    handleAutoRollUpAdvance() {
      // Check if shift is selected before auto-advancing
      if (!this.selectedShift) {
        console.log('AutoRollUp blocked - shift not selected, setting pending auto-advance');
        this.pendingAutoAdvance = true;
        const missingFields = [];
        if (!this.selectedShift) missingFields.push('Shift');
        if (!this.selectedLocation) missingFields.push('Location');
        alert(`Please select ${missingFields.join(' and ')} to continue.`);
        return;
      }
      
      // If shift is selected, proceed with auto-advance
      console.log('AutoRollUp proceeding - shift is selected');
      this.pendingAutoAdvance = false;
      this.$nextTick(() => {
        setTimeout(() => {
          this.nextStep(true); // Skip validation for autoRollUp
        }, 100);
      });
    },

    // Handle shift selection change
    onShiftChange() {
      // Check if there's a pending auto-advance when shift is selected
      if (this.selectedShift && this.pendingAutoAdvance) {
        console.log('Shift selected with pending auto-advance, proceeding to next step');
        this.pendingAutoAdvance = false;
        this.$nextTick(() => {
          setTimeout(() => {
            this.nextStep(true); // Skip validation for autoRollUp
          }, 100);
        });
      }
    },

    // Get the description string for the selected shift
    getSelectedShiftDescription() {
      if (!this.selectedShift) return '';
      
      const selectedShiftOption = this.shiftOptions.find(shift => 
        shift.lookupID === this.selectedShift
      );
      
      return selectedShiftOption ? selectedShiftOption.descr : '';
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
    
    nextStep(skipValidation = false) {
      console.log('nextStep called - isComplexQuestionnaire:', this.isComplexQuestionnaire, 'skipValidation:', skipValidation);
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
            // For checkbox questions, check all selected answers
            if (currentQuestion.questionType === 2) {
              const selectedAnswerIds = this.selectedAnswers[currentQuestion.questionID];
              if (selectedAnswerIds && selectedAnswerIds.length > 0) {
                // Check if any selected answer has a landing page
                for (const answerId of selectedAnswerIds) {
                  const selectedAnswer = currentQuestion.answers.find(a => a.answerID === answerId);
                  if (selectedAnswer && selectedAnswer.toLandingPageID && selectedAnswer.toLandingPageID !== -1) {
                    const landingPage = this.landingPages.find(lp => lp.landingPageID === selectedAnswer.toLandingPageID);
                    if (landingPage) {
                      this.currentLandingPage = landingPage;
                      break; // Use the first landing page found
                    }
                  }
                }
                // If no answer-specific landing page, check if question has a default landing page
                if (!this.currentLandingPage && currentQuestion.toLandingPageID && currentQuestion.toLandingPageID !== -1) {
                  const landingPage = this.landingPages.find(lp => lp.landingPageID === currentQuestion.toLandingPageID);
                  if (landingPage) {
                    this.currentLandingPage = landingPage;
                  }
                }
              }
            } else {
              // Single select logic (existing)
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
          }
          
          // Check if dropdowns are filled before showing results (skip if called from autoRollUp)
          if (!skipValidation && !this.canSubmitSurvey) {
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
          // Start timer immediately
          this.startTimer();
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
          // Check if dropdowns are filled before showing results (skip if called from autoRollUp)
          if (!skipValidation && !this.canSubmitSurvey) {
            console.log('Cannot proceed to results - missing required fields (Shift/Location)');
            // Show validation alert and prevent progression
            const missingFields = [];
            if (!this.selectedShift) missingFields.push('Shift');
            if (!this.selectedLocation) missingFields.push('Location');
            alert(`Please select ${missingFields.join(' and ')} before completing the survey.`);
            return; // Stop progression to results page
          }
          
          // Check if current question has a landing page (for simple questionnaires with landing pages)
          const currentQuestion = this.questions[this.currentStep];
          if (currentQuestion && currentQuestion.toLandingPageID && currentQuestion.toLandingPageID !== -1) {
            const landingPage = this.landingPages.find(lp => lp.landingPageID === currentQuestion.toLandingPageID);
            if (landingPage) {
              this.currentLandingPage = landingPage;
            }
          }
          
          console.log('Reached end of simple questionnaire - showing results');
          this.showResults = true;
          // Start timer immediately
          this.startTimer();
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
      // Check if this is the last question
      if (this.isComplexQuestionnaire) {
        // For complex questionnaires, check if we're at the last accessible question
        const isLastQuestion = this.currentStep === this.accessibleQuestions.length - 1;
        return isLastQuestion ? 'Submit Survey' : 'Continue';
      } else {
        // For simple questionnaires, check if we're at the last question
        const isLastQuestion = this.currentStep === this.questions.length - 1;
        return isLastQuestion ? 'Submit Survey' : 'Continue';
      }
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
            const answerIds = this.selectedAnswers[questionId];
            const question = this.questions.find(q => q.questionID === parseInt(questionId));
            
            if (!question) return null;
            
            // Handle both single select and multiple select questions
            const answerIdArray = Array.isArray(answerIds) ? answerIds : [answerIds];
            
            return answerIdArray
              .map(answerId => {
                const answer = question.answers.find(a => a.answerID === answerId);
                if (answer) {
                  // Format: questionID::questionText::answerText::answerID::rank
                  return `rd${questionId}::${question.text}::${answer.text}::${answerId}::${answer.rank}`;
                }
                return null;
              })
              .filter(item => item !== null)
              .join('~');
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
          Shift: this.getSelectedShiftDescription(), // Use the description string from API
          SurveyId: this.surveyData.surveyID || this.surveyId,
          SurveyAnswers: surveyAnswers,
          SurveyScore: surveyScore,
          SurveyScoreId: 21,
          toLandingPageID: -1,
          toQuestionID: -1,
          SurveyTotalScore: surveyTotalScore,
          StaffName: "",
          LoginName: this.selectedDevice ? this.selectedDevice.customField1 : "", // Use device IP address
          BAcuitySurvey: this.surveyData.bAcuitySurvey || false, // Use the actual bAcuitySurvey value from API response
          surveyAnswersJSON: enhancedQuestionnaire[0] // Add the enhanced questionnaire with user answers (extract string from array)
        };
        
        console.log('Submitting survey data:', payload);
        
        // Submit to original API
        // const response = await fetch('https://demo.galaxycreations.com/gwas/api/Survey/ProcessSurvey', {
        //   method: 'POST',
        //   headers: {
        //     'Content-Type': 'application/json',
        //   },
        //   body: JSON.stringify(payload)
        // });
        
        // if (!response.ok) {
        //   throw new Error(`HTTP error! status: ${response.status}`);
        // }
        
        // const result = await response.json();
        // console.log('Survey submission successful:', result);
        
        // Store the enhanced questionnaire for later access
        this.enhancedQuestionnaire = enhancedQuestionnaire;
        console.log('Enhanced questionnaire with user answers:', this.enhancedQuestionnaire);
        
        // Submit analytics data to AWS API Gateway
        await this.submitAnalyticsData();
        
        this.surveySubmitted = true;
        
        // Show the end screen after successful submission
        this.showResults = true;
        
        // Start the countdown timer
        this.startTimer();
        
        // Show success message to user
        // this.$nextTick(() => {
        //   alert('Survey submitted successfully! Thank you for completing the assessment.');
        // });
        
      } catch (error) {
        console.error('Error submitting survey data:', error);
        
        // Show error flash message
        // this.showErrorFlash();
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
          const selectedAnswerIds = this.selectedAnswers[question.questionID];
          if (selectedAnswerIds) {
            // Handle both single and multiple select as arrays
            const answerIdArray = Array.isArray(selectedAnswerIds) ? selectedAnswerIds : [selectedAnswerIds];
            
            // Create userNavigatedTo object with navigation info (use answer with highest rank)
            let userNavigatedTo = null;
            if (answerIdArray.length > 0) {
              // Get all selected answers
              const selectedAnswers = answerIdArray.map(answerId => 
                question.answers.find(a => a.answerID === answerId)
              ).filter(answer => answer !== undefined);
              
              if (selectedAnswers.length > 0) {
                // Find the answer with the highest rank
                const highestRankAnswer = selectedAnswers.reduce((highest, current) => {
                  const currentRank = current.rank || 0;
                  const highestRank = highest.rank || 0;
                  return currentRank > highestRank ? current : highest;
                });
                
                // Check answer navigation first, but if it's -1 or undefined, fall back to question
                const answerToLandingPageID = highestRankAnswer.toLandingPageID;
                const answerToQuestionID = highestRankAnswer.toQuestionID;
                
                const finalToLandingPageID = (answerToLandingPageID !== undefined && answerToLandingPageID !== -1) 
                  ? answerToLandingPageID 
                  : (question.toLandingPageID !== undefined ? question.toLandingPageID : -1);
                  
                const finalToQuestionID = (answerToQuestionID !== undefined && answerToQuestionID !== -1) 
                  ? answerToQuestionID 
                  : (question.toQuestionID !== undefined ? question.toQuestionID : -1);
                
                userNavigatedTo = {
                  selectedAt: new Date().toISOString(),
                  toLandingPageID: finalToLandingPageID,
                  toQuestionID: finalToQuestionID
                };
              }
            }
            
            // Create Answers object with userSelectedAnswers and userNavigatedTo properties
            enhancedNode.Answers = {
              "userSelectedAnswers": answerIdArray.map(answerId => {
                const selectedAnswer = question.answers.find(a => a.answerID === answerId);
                if (selectedAnswer) {
                  return {
                    ...selectedAnswer // Copy all fields from the original answer
                  };
                }
                return null;
              }).filter(answer => answer !== null),
              "userNavigatedTo": userNavigatedTo
            };
          } else {
            // If no answer was selected, mark it as unanswered
            enhancedNode.Answers = {
              "userSelectedAnswers": [],
              "userNavigatedTo": null
            };
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

    // Submit comprehensive analytics data to AWS API Gateway for Star Schema Database
    async submitAnalyticsData() {
      try {
        console.log('Submitting analytics data to AWS API Gateway with Star Schema structure...');
        
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
        
        // Calculate survey score as completion percentage (0-100)
        const surveyScore = Math.round(completionRate);
        
        // Calculate actual completion time
        const actualCompletionTimeInSeconds = this.getActualCompletionTimeInSeconds();
        const actualCompletionTimeFormatted = this.getActualCompletionTime();
        
        // Prepare question and answer summary data
        const answeredQuestionsList = this.prepareAnsweredQuestionsSummary();
        
        // Build comprehensive questionnaire data for QuickSight
        const questionnairePath = this.buildQuestionnairePath();
        const questionAnalytics = this.buildQuestionAnalytics();
        const enhancedQuestionnairePayload = this.buildEnhancedQuestionnaire();
        
        // Prepare conditional questionnaire metadata
        const conditionalMetadata = this.prepareConditionalMetadata();
        
        // STAR SCHEMA DATA MAPPING
        // Map to your new star schema structure
        
        // 1. SURVEY KEY MAPPING (based on survey_id)
        const surveyKeyMapping = {
          1: 4,  // PHH Patient Classification
          2: 3,  // KPC Patient Classification
          3: 1,  // KPC Stoplight
          4: 5,  // ICU Unit
          5: 6,  // Intake
          6: 2   // Stress First Aid
        };
        
        const surveyKey = surveyKeyMapping[this.surveyData.surveyID || this.surveyId] || 1;
        
        // 2. DEVICE KEY MAPPING
        const deviceKey = this.getDeviceKey(this.selectedLocation, this.selectedDevice);
        
        // 3. SHIFT KEY MAPPING
        const shiftKey = this.getShiftKey(this.getSelectedShiftDescription());
        
        // 4. TIME KEY MAPPING
        const timeKey = this.getTimeKey(submissionDate);
        
        // 5. CALCULATE QUESTION TYPES DISTRIBUTION
        const questionTypesDistribution = this.calculateQuestionTypesDistribution();
        
        // 6. PREPARE BRIDGE TABLE DATA
        const bridgeSubmissionQuestions = this.prepareBridgeSubmissionQuestions(submissionId);
        const bridgeSubmissionAnswers = this.prepareBridgeSubmissionAnswers(submissionId);
        
        // Create comprehensive analytics record for Star Schema
        const analyticsRecord = {
          // === FACT TABLE DATA (fact_survey_submissions) ===
          fact_survey_submissions: {
            submission_id: submissionId,
            survey_key: surveyKey,
            device_key: deviceKey,
            shift_key: shiftKey,
            time_key: timeKey,
            total_questions: totalQuestions,
            questions_shown_to_user: questionsShownToUser,
            answered_questions: answeredQuestions,
            single_select_questions_count: questionTypesDistribution.single_select,
            multiple_select_questions_count: questionTypesDistribution.multiple_select,
            questions_with_multiple_answers: questionTypesDistribution.multiple_select,
            questions_with_single_answer: questionTypesDistribution.single_select,
            unanswered_questions: questionsShownToUser - answeredQuestions,
            question_types_distribution: questionTypesDistribution,
            completion_rate: Math.round(completionRate),
            survey_score: surveyScore,
            time_to_complete_seconds: actualCompletionTimeInSeconds,
            time_to_complete_formatted: actualCompletionTimeFormatted,
            questionnaire_start_time: this.questionnaireStartTime ? this.questionnaireStartTime.toISOString() : null,
            questionnaire_end_time: this.questionnaireEndTime ? this.questionnaireEndTime.toISOString() : null,
            answered_questions_data: answeredQuestionsList,
            questionnaire_path: questionnairePath,
            created_at: timestamp,
            updated_at: timestamp
          },
          
          // === BRIDGE TABLE DATA ===
          bridge_submission_questions: bridgeSubmissionQuestions,
          bridge_submission_answers: bridgeSubmissionAnswers,
          
          // === LEGACY DATA (for backward compatibility) ===
          legacy_data: {
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
            deviceLocation: this.selectedStationText || this.selectedLocation, // Use station name, not code
            deviceId: this.selectedDevice ? this.selectedDevice.stationID : null,
            deviceIP: this.selectedDevice ? this.selectedDevice.customField1 : null,
            shift: this.getSelectedShiftDescription(),
            
            // Survey metrics
            totalQuestions: totalQuestions,
            questionsShownToUser: questionsShownToUser,
            answeredQuestions: answeredQuestions,
            completionRate: Math.round(completionRate),
            surveyScore: surveyScore,
            
            // Time-based analytics
            timeToComplete: actualCompletionTimeInSeconds,
            timeToCompleteFormatted: actualCompletionTimeFormatted,
            questionnaireStartTime: this.questionnaireStartTime ? this.questionnaireStartTime.toISOString() : null,
            questionnaireEndTime: this.questionnaireEndTime ? this.questionnaireEndTime.toISOString() : null,
            isWeekend: [0, 6].includes(new Date().getDay()),
            
            // Technical metadata
            userAgent: navigator.userAgent,
            screenResolution: `${screen.width}x${screen.height}`,
            deviceType: this.getDeviceType(),
            
            // Conditional questionnaire specific fields
            pathIdentifier: conditionalMetadata.pathIdentifier,
            averageQuestionDisplayTime: conditionalMetadata.averageQuestionDisplayTime,
            
            // Summary fields for answered questions (JSON format for flexibility)
            answeredQuestionsData: JSON.stringify(answeredQuestionsList),
            
            // Comprehensive questionnaire data for QuickSight analysis
            questionnairePath: JSON.stringify(questionnairePath),
            questionAnalytics: JSON.stringify(questionAnalytics),
            enhancedQuestionnaireData: enhancedQuestionnairePayload[0],
            
            // Questionnaire structure data
            questionnaireStructure: JSON.stringify({
              survey: this.surveyData,
              totalQuestions: totalQuestions,
              questionTypes: this.questions.map(q => q.type),
              questionRanks: this.questions.map(q => q.rank),
              hasConditionalLogic: this.isComplexQuestionnaire,
              landingPagesCount: this.landingPages ? this.landingPages.length : 0,
              edgesCount: this.edges ? this.edges.length : 0
            })
          }
        };
        
        console.log('Star Schema analytics record:', analyticsRecord);
        
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
          console.log('Star Schema analytics data submitted successfully to AWS');
        } else {
          console.warn('Analytics submission failed:', analyticsResponse.status, analyticsResponse.statusText);
          // Don't throw error to avoid breaking the main submission flow
        }
        
      } catch (error) {
        console.error('Error submitting analytics data:', error);
        // Don't throw error to avoid breaking the main submission flow
      }
    },

    // === HELPER METHODS FOR STAR SCHEMA MAPPING ===

    // Get device key based on location and device info
    getDeviceKey(location) {
      // You'll need to implement device lookup logic
      // This could query your dim_devices table or use a mapping
      const deviceMappings = {
        'ICU iPad': 1001,
        'Med/Surg iPad': 1002,
        'Emergency iPad': 1003,
        'Surgery iPad': 1004,
        'Rehabilitation iPad': 1005
      };
      
      return deviceMappings[location] || 1001; // Default to ICU iPad
    },

    // Get shift key based on shift description
    getShiftKey(shiftDescription) {
      const shiftMappings = {
        'morning': 1,
        'beginning': 1,
        'day': 1,
        'during': 2,
        'afternoon': 2,
        'intervention': 3,
        'evening': 4,
        'night': 5,
        'post traumatic event': 5
      };
      
      const shiftKey = shiftMappings[shiftDescription.toLowerCase()];
      return shiftKey || 1; // Default to beginning shift
    },

    // Get time key based on submission date
    getTimeKey(submissionDate) {
      // You'll need to implement time key lookup logic
      // This could query your dim_time table or calculate based on date
      const dateObj = new Date(submissionDate);
      const dayOfYear = Math.floor((dateObj - new Date(dateObj.getFullYear(), 0, 0)) / (1000 * 60 * 60 * 24));
      
      // Simple time key calculation (you may want to query dim_time instead)
      return 200000 + dayOfYear; // Starting from 200000 to avoid conflicts
    },

    // Calculate question types distribution
    calculateQuestionTypesDistribution() {
      const singleSelectCount = this.questions.filter(q => q.type === 1).length;
      const multipleSelectCount = this.questions.filter(q => q.type === 2).length;
      
      return {
        single_select: singleSelectCount,
        multiple_select: multipleSelectCount
      };
    },

    // Prepare bridge submission questions data
    prepareBridgeSubmissionQuestions(submissionId) {
      const bridgeQuestions = [];
      
      this.questions.forEach((question, index) => {
        // You'll need to get the actual question_key from your dim_questions table
        // For now, we'll use question_id as a placeholder
        const questionKey = question.questionID || question.id;
        
        bridgeQuestions.push({
          submission_id: submissionId,
          question_key: questionKey,
          question_order: index + 1,
          is_answered: this.selectedAnswers[question.questionID || question.id] ? true : false,
          time_spent_seconds: this.getQuestionTimeSpent(question.questionID || question.id)
        });
      });
      
      return bridgeQuestions;
    },

    // Prepare bridge submission answers data
    prepareBridgeSubmissionAnswers(submissionId) {
      const bridgeAnswers = [];
      
      Object.entries(this.selectedAnswers).forEach(([questionId, answerData], index) => {
        // Handle both single and multiple answers
        const answerIds = Array.isArray(answerData) ? answerData : [answerData];
        
        answerIds.forEach(answerId => {
          // selectedAnswers stores answer IDs as numbers, not objects
          // So answerId is already the answer_key we need
          
          // Find the answer object to get navigation info
          const question = this.questions.find(q => q.questionID === parseInt(questionId));
          const answerObject = question ? question.answers.find(a => a.answerID === answerId) : null;
          
          bridgeAnswers.push({
            submission_id: submissionId,
            question_key: parseInt(questionId),
            answer_key: answerId, // Use the answer ID directly
            selection_order: index + 1,
            was_selected: true,
            selection_timestamp: new Date().toISOString(),
            to_question_id: answerObject ? (answerObject.nextQuestionID || null) : null,
            to_landing_page_id: answerObject ? (answerObject.toLandingPageID || null) : null
          });
        });
      });
      
      return bridgeAnswers;
    },

    // Get time spent on a specific question
    getQuestionTimeSpent() {
      // You'll need to implement question timing logic
      // This could track individual question times
      return Math.floor(Math.random() * 60) + 10; // Placeholder: 10-70 seconds
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
  box-sizing: border-box;
}

/* Prevent horizontal overflow on all elements */
* {
  box-sizing: border-box;
}

.therapeutic-container * {
  box-sizing: border-box;
  max-width: 100%;
}

/* Mobile-specific full width fixes */
@media (max-width: 767px) {
  .therapeutic-container {
    padding: 0 8px !important;
    width: 100% !important;
    max-width: 100% !important;
    margin: 0 !important;
    box-sizing: border-box !important;
  }
  
  .main-content-wrapper {
    width: 100% !important;
    max-width: 100% !important;
    padding: 80px 0 20px 0 !important;
    box-sizing: border-box !important;
  }
  
  .question-sanctuary {
    width: 100% !important;
    max-width: 100% !important;
    padding: 0 !important;
    box-sizing: border-box !important;
  }
  
  .answers-container {
    width: 100% !important;
    max-width: 100% !important;
    padding: 0 !important;
    box-sizing: border-box !important;
    gap: 6px !important;
    display: flex !important;
    flex-direction: column !important;
  }
  
  .answer-option {
    width: 100% !important;
    max-width: 100% !important;
    margin: 0 !important;
    padding: 10px 12px !important;
    box-sizing: border-box !important;
    border-radius: 8px !important;
    font-size: 0.8rem !important;
    line-height: 1.3 !important;
    min-height: 45px !important;
    display: flex !important;
    align-items: center !important;
    transition: all 0.2s ease !important;
  }
  
  .enhanced-question-card {
    width: 100% !important;
    max-width: 100% !important;
    margin: 0 !important;
    padding: 0 !important;
    box-sizing: border-box !important;
    border-radius: 12px !important;
    overflow: visible !important;
  }
  
  .survey-header {
    width: 100% !important;
    max-width: 100% !important;
    margin: 0 !important;
    padding: 12px 14px !important;
    box-sizing: border-box !important;
    border-radius: 12px 12px 0 0 !important;
  }
  
  .survey-title {
    font-size: 1.1rem !important;
    font-weight: 700 !important;
    line-height: 1.2 !important;
    margin-bottom: 4px !important;
  }
  
  .survey-subtitle {
    font-size: 0.65rem;
    line-height: 1.2;
    opacity: 0.7;
    font-weight: 300;
  }
  
  .gentle-question {
    font-size: 1.1rem !important;
    font-weight: 600 !important;
    line-height: 1.2 !important;
    margin-bottom: 6px !important;
    color: #2d3436 !important;
  }
  
  .question-description {
    font-size: 0.8rem !important;
    line-height: 1.3 !important;
    color: #636e72 !important;
    margin-bottom: 12px !important;
  }
}

/* Additional mobile styles */
@media (max-width: 767px) {
  .gentle-question,
  .question-description,
  .question-header,
  .question-section {
    width: 100% !important;
    max-width: 100% !important;
    display: block !important;
  }
  
  /* Enhanced mobile card styling */
  .enhanced-question-card {
    background: white !important;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.08) !important;
    border-radius: 12px !important;
    overflow: visible !important;
    margin-bottom: 4px !important;
  }
  
  
  /* Improved mobile header styling */
  .survey-header {
    background: linear-gradient(135deg, #050d9e 0%, #00a0df 100%) !important;
    color: white !important;
    position: relative !important;
  }
  
  .survey-header::after {
    content: '';
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    height: 4px;
    background: linear-gradient(90deg, rgba(255,255,255,0.3) 0%, rgba(255,255,255,0.1) 100%);
  }
  
  /* Better mobile answer options */
  .answer-option {
    background: white !important;
    border: 2px solid transparent !important;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08) !important;
    transition: all 0.3s ease !important;
  }
  
  .answer-option:hover {
    transform: translateY(-2px) !important;
    box-shadow: 0 4px 16px rgba(0, 0, 0, 0.12) !important;
  }
  
  .answer-option.selected {
    border-color: #74b9ff !important;
    box-shadow: 0 4px 16px rgba(116, 185, 255, 0.3) !important;
    transform: translateY(-2px) !important;
  }
}
  
  .survey-header {
    padding: 6px 8px;
    border-radius: 10px;
    margin: 0;
    max-width: 100%;
    overflow: hidden;
  }
  
  
  .rank-tag-chip {
    padding: 2px 4px;
    font-size: 0.4rem;
    align-self: center;
    flex-shrink: 0;
  }
  
  .title-row {
    gap: 4px;
    margin-bottom: 1px;
    flex-wrap: nowrap;
    overflow: hidden;
    flex: 1;
    min-width: 0;
    display: flex !important;
    align-items: center !important;
    flex-direction: row !important;
  }
  
  .header-content {
    display: flex;
    align-items: center;
    gap: 8px;
    width: 100%;
  }
  
  .survey-title-section {
    flex: 1;
    min-width: 0;
    overflow: hidden;
  }
  
  .survey-title {
    font-size: 1rem;
    line-height: 1.1;
    font-weight: 600;
    margin: 0;
    white-space: nowrap;
    flex: 1;
    min-width: 0;
    align-self: center;
  }
  
  .survey-subtitle {
    font-size: 0.6rem;
    margin: 2px 0 0 0;
    line-height: 1.2;
    opacity: 0.7;
    font-weight: 300;
  }
  
  .config-dropdown {
    width: 100%;
    font-size: 0.7rem;
    padding: 6px 8px;
    min-width: 0;
    box-sizing: border-box;
  }
  
  .enhanced-question-card {
    margin: 0;
    max-width: 100%;
  }
  
  .survey-config-section {
    margin: 0 4px;
    padding: 0 4px;
    width: calc(100% - 8px);
    max-width: calc(100% - 8px);
    box-sizing: border-box;
    overflow: visible;
  }
  
  .config-dropdowns {
    display: grid !important;
    grid-template-columns: 1fr 1fr !important;
    gap: 8px !important;
    width: 100%;
    max-width: 100%;
    box-sizing: border-box;
  }
  
  .question-sanctuary {
    margin: 0 4px;
    padding: 8px 4px;
    max-width: calc(100% - 8px);
  }
  
  .answer-option {
    margin: 4px 0;
    padding: 8px 6px;
    font-size: 0.7rem;
    line-height: 1.3;
  }

.therapeutic-container {
  min-height: 100vh;
  width: 100%;
  max-width: 100%;
  margin: 0;
  padding: 0;
  background: linear-gradient(135deg, #00a0df 0%, #1e988a 25%, #a9198d 50%, #050d9e 75%, #f5bd47 100%);
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
  background: linear-gradient(135deg, #00a0df, #050d9e);
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
  padding: 5px 15px 15px 15px;
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

/* Mobile/Tablet Previous Button - Fixed Position Top Left */
.previous-button-mobile-fixed {
  position: fixed;
  top: 15px;
  left: 15px;
  display: none;
  align-items: center;
  gap: 6px;
  padding: 6px 12px;
  border: none;
  border-radius: 18px;
  font-size: 0.7rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.4s ease;
  background: linear-gradient(135deg, rgba(255, 255, 255, 0.95), rgba(255, 255, 255, 0.9));
  color: #495057;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.12);
  border: 1px solid rgba(255, 255, 255, 0.3);
  z-index: 1000;
}

.previous-button-mobile-fixed:hover {
  transform: translateY(-1px);
  box-shadow: 0 6px 15px rgba(0, 0, 0, 0.18);
  background: linear-gradient(135deg, rgba(255, 255, 255, 0.98), rgba(255, 255, 255, 0.95));
}

.previous-button-mobile-fixed .button-icon {
  font-size: 0.8rem;
  font-weight: 600;
}

/* Show mobile button on mobile/tablet, hide desktop button */
@media (max-width: 1199px) {
  .previous-button-mobile-fixed {
    display: flex;
  }
}

/* Smaller mobile screens - adjust button size */
@media (max-width: 575px) {
  .previous-button-mobile-fixed {
    top: 12px;
    left: 12px;
    padding: 5px 10px;
    font-size: 0.65rem;
  }
  
  .previous-button-mobile-fixed .button-icon {
    font-size: 0.7rem;
  }
}

/* Extra small mobile screens */
@media (max-width: 479px) {
  .previous-button-mobile-fixed {
    top: 10px;
    left: 10px;
    padding: 4px 8px;
    font-size: 0.6rem;
  }
  
  .previous-button-mobile-fixed .button-icon {
    font-size: 0.65rem;
  }
}

/* Hide mobile previous button on desktop */
@media (min-width: 1200px) {
  .previous-button-mobile-fixed {
    display: none;
  }
}

/* Rank Tag Chip - Small chip in title row */
.rank-tag-chip {
  position: relative;
  display: inline-block;
  padding: 4px 8px;
  border-radius: 12px;
  font-size: 0.5rem;
  font-weight: 600;
  color: white;
  text-transform: uppercase;
  letter-spacing: 0.3px;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.15);
  z-index: 10;
  animation: tagChipSlideIn 0.4s ease-out;
  border: 1px solid rgba(255, 255, 255, 0.3);
  flex-shrink: 0;
  align-self: center;
}

/* Desktop/Laptop - Rank tag styling */
@media (min-width: 1200px) {
  .rank-tag-chip {
    padding: 5px 10px;
    font-size: 0.55rem;
  }
}

.rank-tag-chip.level-1,
.rank-tag-chip.level-2,
.rank-tag-chip.level-3,
.rank-tag-chip.level-4,
.rank-tag-chip.level-0 {
  background: linear-gradient(135deg, #00b894, #00cec9);
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
  margin: 0;
  line-height: 1.5;
  font-weight: 400;
  text-align: center;
  position: relative;
  top: 0;
  left: 0;
  transform: none;
  padding-left: 0;
}

.question-header {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 0;
}


.question-description {
  color: #95a5a6;
  margin: 0 0 5px 0;
  font-style: italic;
  text-align: center;
  font-size: 0.85rem;
  font-weight: 300;
}

/* Dynamic Question Section */
.question-section {
  text-align: center;
  padding: 0 10px 5px 10px;
  border-radius: 15px;
  transition: background-color 0.3s ease;
  min-height: 120px;
}

.answers-container {
  display: flex;
  flex-direction: column;
  gap: 6px;
  max-width: 600px;
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
  box-shadow: 0 15px 30px rgba(220, 38, 38, 0.3) !important;
}

.answer-option.bg-orange-500.selected {
  background: #f97316 !important;
  background-color: #f97316 !important;
  background-image: none !important;
  color: black !important;
  border: 2px solid #f97316 !important;
  animation: none !important;
  box-shadow: 0 15px 30px rgba(249, 115, 22, 0.3) !important;
}

.answer-option.bg-yellow-400.selected {
  background: #facc15 !important;
  background-color: #facc15 !important;
  background-image: none !important;
  color: black !important;
  border: 2px solid #facc15 !important;
  animation: none !important;
  box-shadow: 0 15px 30px rgba(250, 204, 21, 0.3) !important;
}

.answer-option.bg-green-600.selected {
  background: #16a34a !important;
  background-color: #16a34a !important;
  background-image: none !important;
  color: white !important;
  border: 2px solid #16a34a !important;
  animation: none !important;
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
  background: linear-gradient(135deg, #00a0df, #050d9e);
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

.nav-cloud:disabled:hover {
  transform: none;
  box-shadow: 0 5px 15px rgba(0,0,0,0.1);
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
  background: linear-gradient(135deg, #00a0df, #050d9e);
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

/* Mobile: Ensure results page covers entire screen */
@media (max-width: 768px) {
  .healing-results {
    position: absolute !important;
    top: 0 !important;
    left: 0 !important;
    width: 100% !important;
    height: 100% !important;
    z-index: 9999 !important;
    background: rgba(0,0,0,0.9) !important;
    display: flex !important;
    align-items: center !important;
    justify-content: center !important;
  }
  
  /* Ensure therapeutic container is positioned relative for absolute positioning to work */
  .therapeutic-container {
    position: relative !important;
  }
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
  background: linear-gradient(135deg, #00a0df, #050d9e);
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


/* Dynamic Continue button styling */
.continue-button-after-selected {
  display: flex;
  justify-content: flex-end;
  margin: 2px 0;
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
  background: linear-gradient(135deg, #00a0df, #050d9e);
  color: white;
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
}

.continue-button-dynamic:hover {
  transform: translateY(-1px);
  box-shadow: 0 6px 20px rgba(0,0,0,0.25);
}

/* Checkbox styling for multiple select questions */
.checkbox-mode {
  position: relative;
}

.checkbox-container {
  position: absolute;
  left: 15px;
  top: 50%;
  transform: translateY(-50%);
  z-index: 2;
}

.answer-checkbox {
  width: 20px;
  height: 20px;
  cursor: pointer;
  accent-color: #667eea;
}

.checkbox-mode .answer-content {
  padding-left: 50px;
}

.checkbox-mode .answer-text {
  margin-left: 0;
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
    padding: 0;
    max-width: 100%;
    margin: 0;
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
  
  
  .config-dropdowns {
    justify-content: center;
    max-width: 100%;
    margin: 0;
    gap: 5px;
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
    transition: background, border-color, transform, box-shadow, color 0.3s cubic-bezier(0.4, 0, 0.2, 1);
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
    transition-property: background, border-color, transform, box-shadow, color !important;
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
  
  
  .circular-progress-content {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    text-align: center;
    color: white;
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
  
  .main-content-wrapper {
    padding: 100px 0 30px 0 !important;
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
    padding: 5px 18px 20px 18px;
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
    gap: 5px;
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
  
}

/* Tablet Portrait (576px to 767px) - Balanced Spacing */
@media (max-width: 767px) and (min-width: 576px) {
  .therapeutic-container {
    padding: 0 12px 0 12px;
    min-height: 100vh;
  }
  
  .main-content-wrapper {
    padding: 90px 0 20px 0 !important;
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
  
  
  
  
  .question-sanctuary {
    max-width: 100%;
    margin: 0 auto 15px;
  }
  
  .progress-border-container {
    margin: 8px;
  }
  
  .question-cloud {
    padding: 5px 20px 16px 20px;
    min-height: 320px;
    border-radius: 18px;
  }
  
  .rank-tag-chip {
    padding: 5px 10px;
    font-size: 0.65rem;
  }
  
  .question-section {
    padding: 0 8px 5px 8px;
    min-height: 240px;
  }
  
  .question-header {
    margin: 0;
  }
  
  .gentle-question {
    font-size: 1.2rem;
    margin: 0;
    top: 0;
  }
  
  .question-description {
    font-size: 0.8rem;
    margin: 0 0 3px 0;
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
  
}

/* Mobile Landscape (480px to 575px) - Balanced Spacing */
@media (max-width: 575px) and (min-width: 480px) {
  .therapeutic-container {
    padding: 0 7px !important;
    min-height: 100vh;
    width: 100% !important;
    max-width: 100% !important;
    box-sizing: border-box !important;
  }
  
  .main-content-wrapper {
    padding: 85px 0 20px 0 !important;
    box-sizing: border-box !important;
  }
  
  .question-sanctuary {
    padding: 0 !important;
    box-sizing: border-box !important;
  }
  
  .enhanced-question-card {
    padding: 0 !important;
    box-sizing: border-box !important;
  }
  
  .survey-header {
    padding: 11px 12px !important;
    box-sizing: border-box !important;
  }
  
  .answer-option {
    padding: 9px 12px !important;
    margin: 0 !important;
    box-sizing: border-box !important;
    min-height: 42px !important;
    font-size: 0.78rem !important;
  }
  
  .survey-title {
    font-size: 1.05rem !important;
  }
  
  .gentle-question {
    font-size: 1.05rem !important;
  }
  
  .config-dropdown {
    width: 100% !important;
    padding: 7px 9px !important;
    font-size: 0.78rem !important;
    min-height: 38px !important;
    min-width: 0 !important;
    box-sizing: border-box !important;
  }
  
  .survey-config-section {
    padding: 0 7px !important;
    margin-bottom: 7px !important;
  }
  
  .config-dropdowns {
    gap: 5px !important;
    width: 100% !important;
    max-width: 100% !important;
    box-sizing: border-box !important;
  }
  
  .answers-container {
    gap: 5px !important;
  }
  
  .question-description {
    margin-bottom: 10px !important;
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
    padding: 3px 15px 12px 15px;
    min-height: 280px;
    border-radius: 12px;
    flex: 1;
    display: flex;
    flex-direction: column;
  }
  
  .rank-tag-chip {
    padding: 4px 8px;
    font-size: 0.6rem;
  }
  
  .question-section {
    flex: 1;
    display: flex;
    flex-direction: column;
    padding: 0 6px 5px 6px;
    min-height: 200px;
  }
  
  .question-header {
    margin: 0;
  }
  
  .gentle-question {
    font-size: 1.1rem;
    margin: 0;
    top: 0;
  }
  
  .question-description {
    font-size: 0.75rem;
    margin: 0 0 2px 0;
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
    padding: 0 6px !important;
    min-height: 100vh;
    width: 100% !important;
    max-width: 100% !important;
    box-sizing: border-box !important;
  }
  
  .main-content-wrapper {
    padding: 80px 0 20px 0 !important;
    box-sizing: border-box !important;
  }
  
  .question-sanctuary {
    padding: 0 !important;
    box-sizing: border-box !important;
  }
  
  .enhanced-question-card {
    padding: 0 !important;
    box-sizing: border-box !important;
  }
  
  .survey-header {
    padding: 10px 10px !important;
    box-sizing: border-box !important;
  }
  
  .answer-option {
    padding: 8px 10px !important;
    margin: 0 !important;
    box-sizing: border-box !important;
    min-height: 40px !important;
    font-size: 0.75rem !important;
  }
  
  .survey-title {
    font-size: 1.0rem !important;
  }
  
  .gentle-question {
    font-size: 1.0rem !important;
  }
  
  .config-dropdown {
    width: 100% !important;
    padding: 6px 8px !important;
    font-size: 0.75rem !important;
    min-height: 36px !important;
    min-width: 0 !important;
    box-sizing: border-box !important;
  }
  
  .survey-config-section {
    padding: 0 6px !important;
    margin-bottom: 6px !important;
  }
  
  .config-dropdowns {
    gap: 4px !important;
    width: 100% !important;
    max-width: 100% !important;
    box-sizing: border-box !important;
  }
  
  .answers-container {
    gap: 4px !important;
  }
  
  .question-description {
    margin-bottom: 8px !important;
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
    padding: 2px 12px 10px 12px;
    min-height: 250px;
    border-radius: 10px;
    flex: 1;
    display: flex;
    flex-direction: column;
  }
  
  .rank-tag-chip {
    padding: 3px 6px;
    font-size: 0.55rem;
  }
  
  .question-section {
    flex: 1;
    display: flex;
    flex-direction: column;
    padding: 0 4px 3px 4px;
    min-height: 160px;
  }
  
  .question-header {
    margin: 0;
  }
  
  .gentle-question {
    font-size: 1rem;
    margin: 0;
    top: 0;
  }
  
  .question-description {
    font-size: 0.7rem;
    margin: 0 0 1px 0;
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
  
}

/* Disabled state for answer options when location not selected */
.answer-option.disabled {
  opacity: 0.5;
  cursor: not-allowed;
  pointer-events: none;
  filter: grayscale(0.3);
}

.answer-option.disabled:hover {
  transform: none !important;
  box-shadow: none !important;
}

/* Device selection message overlay */
.device-selection-message {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 10;
  text-align: center;
}

.device-selection-message .message-text {
  background: rgba(255, 255, 255, 0.9);
  color: #e74c3c;
  font-size: 1rem;
  font-weight: 500;
  padding: 8px 16px;
  border-radius: 6px;
  border: 1px solid rgba(231, 76, 60, 0.3);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

/* Disabled container positioning */
.answers-container.disabled-container {
  position: relative;
}

/* Touch-friendly improvements for mobile devices */
@media (hover: none) and (pointer: coarse) {
  .answer-option {
    min-height: 48px; /* Minimum touch target size */
    padding: 12px 16px;
    font-size: 0.9rem;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    border: 1px solid rgba(0, 0, 0, 0.05);
  }
  
  .answer-option:active {
    transform: scale(0.98);
    box-shadow: 0 1px 4px rgba(0, 0, 0, 0.15);
  }
  
  .config-dropdown {
    min-height: 44px;
    padding: 8px 12px;
    font-size: 0.85rem;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  }
  
  .config-dropdown:active {
    transform: scale(0.98);
  }
  
  .gentle-question {
    font-size: 1.1rem;
    line-height: 1.4;
  }
  
  .question-description {
    font-size: 0.9rem;
    margin-bottom: 20px;
  }
  
  /* Enhanced visual feedback for touch */
  .answer-option.selected {
    transform: scale(1.02);
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  }
}

/* Additional touch-friendly styles */
.device-selection-message .message-text {
  font-size: 0.9rem;
}

/* Touch device optimizations */
@media (hover: none) and (pointer: coarse) {
  .nav-cloud,
  .support-button,
  .continue-button-dynamic,
  .retry-button {
    min-height: 44px; /* Minimum touch target size */
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
}

/* High DPI displays */
@media (-webkit-min-device-pixel-ratio: 2), (min-resolution: 192dpi) {
  .breath-circle,
  .floating-circle {
    border-width: 1px; /* Thinner borders on high DPI */
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
    padding: 3px 20px 15px 20px;
    min-height: 250px;
  }
  
  .gentle-question {
    font-size: 1rem;
    margin-bottom: 15px;
  }
  
  .question-description {
    font-size: 0.75rem;
    margin-bottom: 15px;
  }
  
  .answer-option {
    padding: 10px;
    margin-bottom: 6px;
  }
  
  .answer-text {
    font-size: 0.8rem;
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
  
}


/* Enhanced Question Card Styles */
.enhanced-question-card {
  max-width: 100%;
  margin: 0;
  background: #ffffff;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.06);
  border: 1px solid #e9ecef;
  overflow: visible;
  position: relative;
  max-height: none;
  display: flex;
  flex-direction: column;
}

/* Survey Configuration Section */
.survey-config-section {
  padding: 12px 20px;
  background: #f8f9fa;
  border-bottom: 1px solid #e9ecef;
  flex-shrink: 0;
  overflow: visible;
  position: relative;
  z-index: 100;
}

.survey-config-section .config-dropdowns {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 2px;
  margin: 12px 0;
  width: 100%;
}

.survey-config-section .config-dropdown {
  padding: 8px 12px;
  border: 1px solid #dee2e6;
  border-radius: 6px;
  background: white;
  font-size: 0.9rem;
  color: #495057;
  flex: 1;
  transition: all 0.3s ease;
}

.survey-config-section .config-dropdown:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 2px rgba(102, 126, 234, 0.1);
}

.survey-config-section .config-dropdown.required-field {
  border-color: #dc3545;
}

.survey-header {
  background: linear-gradient(135deg, #050d9e 0%, #00a0df 50%, #a9198d 100%);
  padding: 18px 25px;
  color: white;
  position: relative;
  flex-shrink: 0;
  border-radius: 20px;
  box-shadow: 0 10px 30px rgba(102, 126, 234, 0.3);
  overflow: hidden;
}

/* Logo Section Styles */
.sfa-logo-section, .nj-logo-section {
  display: flex;
  align-items: center;
  flex-shrink: 0;
}

.nj-logo-section {
  margin-left: auto; /* Ensure it stays on the right */
  justify-content: flex-end;
}

.sfa-logo {
  height: 40px;
  width: auto;
  object-fit: contain;
  transition: transform 0.3s ease;
}

.nj-logo {
  height: 60px; /* Made larger */
  width: auto;
  object-fit: contain;
  transition: transform 0.3s ease;
}

.sfa-logo:hover, .nj-logo:hover {
  transform: scale(1.05);
}

/* Responsive logo adjustments */
@media (max-width: 768px) {
  .sfa-logo {
    height: 30px;
  }
  .nj-logo {
    height: 45px; /* Larger on mobile too */
  }
}

@media (max-width: 480px) {
  .sfa-logo {
    height: 25px;
  }
  .nj-logo {
    height: 35px; /* Larger on small mobile too */
  }
}

/* Desktop - Full width header */
@media (min-width: 1200px) {
  .survey-header {
    width: 100%;
    border-radius: 20px;
    padding: 15px 30px;
  }
  
  .enhanced-question-card {
    margin: 5px 0;
    max-width: 100%;
    border-radius: 20px;
  }
  
  .therapeutic-container {
    padding: 5px 30px;
  }
}

.survey-header::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(135deg, rgba(255,255,255,0.1) 0%, rgba(255,255,255,0.05) 100%);
  pointer-events: none;
}

.header-content {
  display: flex;
  flex-direction: row;
  align-items: center;
  position: relative;
  z-index: 2;
  gap: 20px;
  width: 100%;
}

.survey-title-section {
  flex: 1;
  text-align: left;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}

.title-row {
  display: flex;
  align-items: center;
  justify-content: flex-start;
  gap: 12px;
  margin-bottom: 4px;
}

.survey-title {
  font-size: 1.2rem;
  font-weight: 700;
  margin: 0;
  letter-spacing: -0.5px;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  background: linear-gradient(45deg, #ffffff, #f0f8ff);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  align-self: center;
}

/* Desktop title and subtitle - ensure they override mobile styles */
@media (min-width: 1200px) {
  .survey-title {
    font-size: 1.4rem !important;
    font-weight: 700 !important;
  }
  
  .survey-subtitle {
    font-size: 0.7rem !important;
    margin: 0 !important;
    opacity: 0.95 !important;
    font-weight: 400 !important;
    line-height: 1.3 !important;
    text-shadow: 0 1px 2px rgba(0, 0, 0, 0.1) !important;
    color: rgba(255, 255, 255, 0.95) !important;
  }
}

/* Tablet subtitle - medium size */
@media (min-width: 768px) and (max-width: 1199px) {
  .survey-subtitle {
    font-size: 0.68rem !important;
    margin: 0 !important;
    opacity: 0.85 !important;
    font-weight: 350 !important;
    line-height: 1.3 !important;
    text-shadow: 0 1px 2px rgba(0, 0, 0, 0.1) !important;
    color: rgba(255, 255, 255, 0.95) !important;
  }
}

.survey-subtitle {
  font-size: 0.7rem;
  margin: 0;
  opacity: 0.95;
  font-weight: 400;
  line-height: 1.3;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
  color: rgba(255, 255, 255, 0.95);
}





/* Enhanced Question Sanctuary */
.enhanced-question-card .question-sanctuary {
  padding: 15px;
  background: transparent;
  border-radius: 0;
  box-shadow: none;
  margin: 0;
  max-width: none;
  flex: 1;
  overflow-y: auto;
}

/* Compact spacing for enhanced card content */
.enhanced-question-card .question-cloud {
  margin-bottom: 15px;
}

.enhanced-question-card .question-text {
  font-size: 1.3rem;
  margin-bottom: 8px;
}

.enhanced-question-card .question-instruction {
  font-size: 0.9rem;
  margin-bottom: 15px;
}

.enhanced-question-card .answer-option {
  margin-bottom: 12px;
  padding: 15px 25px;
  white-space: normal;
  word-wrap: break-word;
  line-height: 1.4;
  width: 100% !important;
  max-width: none !important;
  box-sizing: border-box;
  min-width: 0;
  flex: 1;
}

.enhanced-question-card .continue-button {
  margin-top: 15px;
  padding: 10px 20px;
}

/* Enhanced card display */
.enhanced-question-card {
  display: block;
  max-width: 100%;
  margin: 0;
}

/* Center the main content wrapper */
.main-content-wrapper {
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  align-items: center;
  min-height: auto;
  gap: 20px;
  padding-top: 20px;
  padding-bottom: 20px;
}
  
/* Supportive navigation */
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

/* Submission Status Styles */
.submission-status {
  margin: 5px 0;
  padding: 5px;
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
  gap: 5px;
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

/* Circular Timer Styles */
.circular-timer-container {
  margin-top: 5px;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 5px;
}

/* Results Top Actions - Take Assessment Again Button */
.results-top-actions {
  margin: 5px 0;
  display: flex;
  justify-content: center;
  align-items: center;
}

/* Timer Top Actions - Take Assessment Again Button */
.timer-top-actions {
  margin-bottom: 20px;
  display: flex;
  justify-content: center;
  align-items: center;
}

.circular-timer {
  position: relative;
  width: 120px;
  height: 120px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.timer-svg {
  width: 100%;
  height: 100%;
  transform: rotate(-90deg);
}

.timer-content {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
  z-index: 2;
}

.timer-number {
  font-size: 2rem;
  font-weight: bold;
  color: #2d3436;
  line-height: 1;
  margin-bottom: 2px;
}

.timer-label {
  font-size: 0.7rem;
  color: #636e72;
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

/* Flash Message Styles */
.flash-message {
  position: relative;
  max-width: 320px;
  margin: 0 auto;
  animation: flashSlideIn 0.5s ease-out;
}

.flash-content {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 16px 20px;
  border-radius: 12px;
  position: relative;
  overflow: hidden;
}

.flash-content.success {
  background: linear-gradient(135deg, #00a0df, #050d9e);
  box-shadow: 0 4px 20px rgba(116, 185, 255, 0.3);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.flash-content.error {
  background: linear-gradient(135deg, #e17055, #d63031);
  box-shadow: 0 4px 20px rgba(214, 48, 49, 0.3);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.flash-content::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
  animation: flashShimmer 2s infinite;
}

.flash-icon {
  font-size: 1.5rem;
  flex-shrink: 0;
  animation: flashPulse 1s ease-in-out infinite alternate;
}

.flash-text {
  flex: 1;
  color: white;
}

.flash-title {
  margin: 0;
  font-weight: 600;
  font-size: 0.9rem;
}

.timer-message {
  margin-top: 5px;
  color: #636e72;
  font-size: 0.9rem;
}

/* Flash Message Styles */
.flash-message {
  position: relative;
  max-width: 320px;
  margin: 0 auto;
  animation: flashSlideIn 0.5s ease-out;
}

.flash-content {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 16px 20px;
  border-radius: 12px;
  position: relative;
  overflow: hidden;
}

.flash-content.success {
  background: linear-gradient(135deg, #00a0df, #050d9e);
  box-shadow: 0 4px 20px rgba(116, 185, 255, 0.3);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.flash-content.error {
  background: linear-gradient(135deg, #e17055, #d63031);
  box-shadow: 0 4px 20px rgba(214, 48, 49, 0.3);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.flash-content::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
  animation: flashShimmer 2s infinite;
}

.flash-icon {
  font-size: 1.5rem;
  flex-shrink: 0;
  animation: flashPulse 1s ease-in-out infinite alternate;
}

.flash-text {
  flex: 1;
  color: white;
}

.flash-title {
  font-size: 0.9rem;
  font-weight: bold;
  margin: 0 0 4px 0;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
}

.flash-subtitle {
  font-size: 0.8rem;
  margin: 0;
  opacity: 0.9;
  font-weight: 500;
}

/* Flash Message Animations */
@keyframes flashSlideIn {
  0% {
    opacity: 0;
    transform: translateY(-20px) scale(0.9);
  }
  100% {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

@keyframes flashPulse {
  0% {
    transform: scale(1);
  }
  100% {
    transform: scale(1.1);
  }
}

@keyframes flashShimmer {
  0% {
    left: -100%;
  }
  100% {
    left: 100%;
  }
}

/* Timer Message Styles */
.timer-message {
  margin-top: 5px;
  text-align: center;
}

.timer-message p {
  font-size: 0.9rem;
  color: #636e72;
  margin: 0;
  padding: 10px 16px;
  background: linear-gradient(135deg, #f8f9fa, #e9ecef);
  border-radius: 20px;
  border: 1px solid #dee2e6;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  display: inline-block;
}

/* Animation for the timer number */
.timer-number {
  animation: timerPulse 1s ease-in-out infinite alternate;
}

@keyframes timerPulse {
  0% { 
    transform: scale(1);
    color: #2d3436;
  }
  100% { 
    transform: scale(1.05);
    color: #74b9ff;
  }
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

.config-dropdowns {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 2px;
  width: 100%;
  max-width: 100%;
  box-sizing: border-box;
}

/* Basic dropdown styles - will be overridden by media queries */
.config-dropdown {
  min-width: 0;
  background: rgba(255, 255, 255, 0.9);
  border: 2px solid rgba(45, 52, 54, 0.2);
  border-radius: 10px;
  padding: 8px 12px;
  font-size: 0.85rem;
  color: #2d3436;
  backdrop-filter: blur(10px);
  transition: background, border-color, transform, box-shadow, color 0.3s ease;
  box-sizing: border-box;
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
  transition-property: background, border-color, transform, box-shadow, color !important;
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

/* Prevent background-image transitions on all dropdown states */
.config-dropdown,
.config-dropdown:hover,
.config-dropdown:focus,
.config-dropdown.required-field,
.config-dropdown.required-field:hover,
.config-dropdown.required-field:focus {
  transition: background, border-color, transform, box-shadow, color 0.3s ease !important;
  transition-property: background, border-color, transform, box-shadow, color !important;
}

/* Custom dropdown styles */
.custom-dropdown-container {
  position: relative;
  min-width: 0;
  z-index: 1000;
}


.regular-select-container {
  min-width: 0;
}

.custom-dropdown {
  display: flex;
  justify-content: space-between;
  align-items: center;
  user-select: none;
  width: 100%;
}

.custom-dropdown .dropdown-arrow {
  transition: transform 0.3s ease;
  width: 16px;
  height: 16px;
  color: #636e72;
  filter: drop-shadow(0 1px 2px rgba(0,0,0,0.1));
}

.custom-dropdown.dropdown-open .dropdown-arrow {
  transform: rotate(180deg);
}

.custom-dropdown:hover .dropdown-arrow {
  color: #4a5568;
  filter: drop-shadow(0 2px 4px rgba(0,0,0,0.15));
}

.dropdown-options {
  position: absolute;
  top: 100%;
  left: 0;
  width: 100%;
  background: white;
  border: 2px solid #e9ecef;
  border-top: none;
  border-radius: 0 0 10px 10px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  z-index: 99999 !important;
  max-height: 200px;
  overflow-y: auto;
  box-sizing: border-box;
}

.dropdown-option {
  padding: 12px 16px;
  cursor: pointer;
  transition: background-color 0.2s ease;
  border-bottom: 1px solid #f8f9fa;
}

.dropdown-option:hover {
  background-color: #f8f9fa;
}

.dropdown-option:last-child {
  border-bottom: none;
}

/* Mobile First - Ensure row layout on all mobile devices */
@media (max-width: 1199px) {
  .survey-config-section {
    padding: 0 8px;
    margin-bottom: 8px;
    max-width: 100% !important;
    box-sizing: border-box !important;
  }
  
  .config-dropdowns {
    display: grid !important;
    grid-template-columns: 1fr 1fr !important;
    gap: 8px !important;
    width: 100% !important;
    max-width: 100% !important;
    box-sizing: border-box !important;
    margin: 10px 0 !important;
  }
  
  .config-dropdown {
    flex: 1 !important;
    width: 100% !important;
    padding: 8px 10px !important;
    font-size: 0.8rem !important;
    border-radius: 8px !important;
    border: 2px solid #e9ecef !important;
    background: white !important;
    min-height: 40px !important;
    box-sizing: border-box !important;
    transition: background, border-color, transform, box-shadow, color 0.2s ease !important;
    transition-property: background, border-color, transform, box-shadow, color !important;
  }
  
  .config-dropdown:focus {
    border-color: #74b9ff !important;
    outline: none !important;
    box-shadow: 0 0 0 3px rgba(116, 185, 255, 0.1) !important;
  }
  
  .config-dropdown.required-field {
    border-color: #e74c3c !important;
  }
}

/* Additional mobile configuration styles */
@media (max-width: 1199px) {
  /* Show linear progress on smaller screens */
  .circular-progress-container {
    display: none;
  }
  
  .linear-progress-container {
    display: block;
  }
  
  .config-dropdowns {
    display: grid !important;
    grid-template-columns: 1fr 1fr !important;
    gap: 8px !important;
    margin: 8px 0 !important;
  }
  
  .config-dropdown {
    flex: 1 !important;
    width: 100% !important;
    padding: 6px 8px;
    font-size: 0.75rem;
  }
}

@media (max-width: 575px) {
  .survey-config-section {
    padding: 0 10px;
    margin-bottom: 12px;
  }
  
  .config-dropdowns {
    display: grid !important;
    grid-template-columns: 1fr 1fr !important;
    gap: 8px !important;
    margin: 6px 0 !important;
  }
  
  .config-dropdown {
    flex: 1 !important;
    width: 100% !important;
    padding: 5px 6px;
    font-size: 0.7rem;
  }
}

@media (max-width: 479px) {
  .survey-config-section {
    padding: 0 8px;
    margin-bottom: 10px;
  }
  
  .config-dropdowns {
    display: grid !important;
    grid-template-columns: 1fr 1fr !important;
    gap: 8px !important;
    margin: 5px 0 !important;
  }
  
  .dropdown-group {
    flex: 1 !important;
    width: 100% !important;
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
