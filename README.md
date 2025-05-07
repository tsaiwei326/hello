# helloimport 
React from 'react';

const MeaslesExposureBot = () => {
  const [state, setState] = React.useState({
    step: 0,
    chatHistory: [{
      sender: 'bot',
      message: "Welcome to the Measles Exposure Management Assistant. I'll help guide management decisions for patients with measles exposure."
    }],
    userInput: '',
    patientData: {
      age: null,
      vaccineStatus: null,
      immunityStatus: null,
      pregnant: null,
      immunocompromised: null,
      exposureTime: null,
      highriskSetting: null
    },
    assessmentComplete: false
  });

  // Questions for gathering patient information
  const questions = [
    {
      id: 'age',
      text: "What is the patient's age?",
      options: ["<6 months", "6-11 months", "1-4 years", "5-18 years", "19-60 years", "61+ years"]
    },
    {
      id: 'vaccineStatus',
      text: "What is the patient's measles vaccination status?",
      options: [
        "0 doses",
        "1 dose",
        "2 or more doses",
        "Unknown"
      ]
    },
    {
      id: 'immunityStatus',
      text: "Does the patient have documented immunity to measles? (laboratory evidence of immunity or laboratory confirmation of disease)",
      options: ["Yes", "No", "Unknown"]
    },
    {
      id: 'pregnant',
      text: "Is the patient pregnant?",
      options: ["Yes", "No", "Unknown/Not applicable"]
    },
    {
      id: 'immunocompromised',
      text: "Is the patient severely immunocompromised? This includes: Primary immunodeficiency, within 2 months after solid organ transplant, HIV with CD4 <200, Steroids ≥20mg/day for >2 weeks, receiving immune modulators such as rituximab, or post-bone marrow transplant",
      options: ["Yes", "No"]
    },
    {
      id: 'exposureTime',
      text: "How long has it been since measles exposure?",
      options: [
        "Less than 72 hours",
        "3-6 days post-exposure",
        "More than 6 days post-exposure"
      ]
    },
    {
      id: 'highriskSetting',
      text: "Is the patient in a high-risk setting? (healthcare facility, school, childcare facility, congregate living)",
      options: ["Yes", "No"]
    }
  ];

  // Process user input from dropdown
  const handleSubmit = () => {
    if (!state.userInput) return;
   
    // Add user message to chat history
    const newHistory = [...state.chatHistory, { sender: 'user', message: state.userInput }];
   
    // Process the user's answer
    const currentQ = questions[state.step];
    let processedAnswer = state.userInput;
   
    // Special processing for age to convert to numeric values for calculations
    if (currentQ.id === 'age') {
      if (processedAnswer === "<6 months") {
        processedAnswer = 0.25; // approximate 3 months
      } else if (processedAnswer === "6-11 months") {
        processedAnswer = 0.75; // approximate 9 months
      } else if (processedAnswer === "1-4 years") {
        processedAnswer = 2.5; // middle of range
      } else if (processedAnswer === "5-18 years") {
        processedAnswer = 12; // middle of range
      } else if (processedAnswer === "19-60 years") {
        processedAnswer = 40; // middle of range
      } else if (processedAnswer === "61+ years") {
        processedAnswer = 65; // approximate
      }
    }
   
    // Update patient data
    const newPatientData = {...state.patientData};
    newPatientData[currentQ.id] = processedAnswer;
   
    // Move to next question or generate recommendation
    if (state.step < questions.length - 1) {
      const nextQuestion = questions[state.step + 1];
      let botResponse = nextQuestion.text;
      
      newHistory.push({ sender: 'bot', message: botResponse });
     
      setState({
        ...state,
        step: state.step + 1,
        chatHistory: newHistory,
        userInput: '',
        patientData: newPatientData
      });
    } else {
      // All questions answered, generate recommendation
      const recommendation = generateRecommendation(newPatientData);
      newHistory.push({ sender: 'bot', message: recommendation });
     
      setState({
        ...state,
        chatHistory: newHistory,
        userInput: '',
        patientData: newPatientData,
        assessmentComplete: true
      });
    }
  };

  // Function to generate clinical recommendations based on patient data
  const generateRecommendation = (data) => {
    let recommendation = "## Measles Post-Exposure Management Recommendation\n\n";
    recommendation += "### Patient Assessment:\n";
    recommendation += `- Age: ${formatAge(data.age)}\n`;
    recommendation += `- Measles Vaccination: ${data.vaccineStatus}\n`;
    recommendation += `- Documented Immunity: ${data.immunityStatus}\n`;
    recommendation += `- Pregnancy Status: ${data.pregnant}\n`;
    recommendation += `- Severely Immunocompromised: ${data.immunocompromised}\n`;
    recommendation += `- Time Since Exposure: ${data.exposureTime}\n`;
    recommendation += `- High-Risk Setting: ${data.highriskSetting}\n\n`;
   
    recommendation += "### Recommended Management:\n\n";
   
    // Determine immunity status
    const presumedImmune =
      data.immunityStatus === "Yes" ||
      (data.vaccineStatus === "2 or more doses" && data.immunocompromised === "No") ||
      (data.age >= 61); // Born before 1957 presumed immune
   
    // MMR contraindications
    const mmrContraindicated =
      data.pregnant === "Yes" ||
      data.immunocompromised === "Yes" ||
      data.age < 1;
   
    // IVIG indications
    const ivigCandidate =
      data.pregnant === "Yes" ||
      data.immunocompromised === "Yes" ||
      data.age < 1;
   
    // Post-exposure prophylaxis recommendations
    if (presumedImmune) {
      recommendation += "This patient is **presumed immune** to measles based on their history. No post-exposure prophylaxis is indicated.\n\n";
      recommendation += "- Advise to monitor for symptoms for 21 days post-exposure\n";
      recommendation += "- If symptoms develop, isolate and test immediately\n";
    } else {
      // Not immune - needs intervention
      if (data.exposureTime === "Less than 72 hours") {
        if (!mmrContraindicated) {
          recommendation += "**Recommended: MMR vaccination**\n\n";
          recommendation += "- Administer MMR vaccine within 72 hours of exposure\n";
          recommendation += "- May prevent or modify disease course if given within this timeframe\n";
         
          if (data.age >= 1 && data.age < 6 && data.vaccineStatus === "0 doses") {
            recommendation += "- For this patient (aged 1-5 years), this would be their first MMR dose\n";
          } else if (data.age >= 6 && data.vaccineStatus === "1 dose") {
            recommendation += "- For this patient (aged 6+ with 1 prior dose), this would complete their MMR series\n";
          }
        } else {
          recommendation += "**MMR vaccine is contraindicated for this patient.**\n\n";
        }
       
        if (ivigCandidate) {
          recommendation += "**Recommended: Immune Globulin (IG)**\n\n";
          recommendation += "- Administer IG within 6 days of exposure\n";
         
          if (data.immunocompromised === "Yes") {
            recommendation += "- For severely immunocompromised patients: IGIV at 400 mg/kg\n";
          } else if (data.pregnant === "Yes") {
            recommendation += "- For pregnant patients: IGIV at 400 mg/kg\n";
          } else if (data.age < 1) {
            if (data.age < 0.5) { // < 6 months
              recommendation += "- For infants <6 months: IGIM at 0.5 mL/kg (max 15 mL)\n";
            } else { // 6-11 months
              recommendation += "- For infants 6-11 months: Consider MMR vaccine instead if no contraindications exist\n";
              recommendation += "- If MMR contraindicated: IGIM at 0.5 mL/kg (max 15 mL)\n";
            }
          }
        }
      } else if (data.exposureTime === "3-6 days post-exposure") {
        if (ivigCandidate) {
          recommendation += "**Recommended: Immune Globulin (IG)**\n\n";
          recommendation += "- Administer IG within 6 days of exposure\n";
         
          if (data.immunocompromised === "Yes") {
            recommendation += "- For severely immunocompromised patients: IGIV at 400 mg/kg\n";
          } else if (data.pregnant === "Yes") {
            recommendation += "- For pregnant patients: IGIV at 400 mg/kg\n";
          } else if (data.age < 1) {
            if (data.age < 0.5) { // < 6 months
              recommendation += "- For infants <6 months: IGIM at 0.5 mL/kg (max 15 mL)\n";
            } else { // 6-11 months
              recommendation += "- For infants 6-11 months: IGIM at 0.5 mL/kg (max 15 mL)\n";
            }
          }
        } else if (!mmrContraindicated) {
          recommendation += "**Note:** MMR vaccine is less effective when given >72 hours post-exposure but can still be considered for susceptible individuals with ongoing or future risk.\n\n";
          recommendation += "- Consider Immune Globulin (0.5 mL/kg, max 15 mL) if within 6 days of exposure\n";
        }
      } else {
        // > 6 days post-exposure
        recommendation += "**Post-exposure prophylaxis is no longer indicated** (>6 days since exposure).\n\n";
        recommendation += "- Monitor for symptoms through 21 days post-exposure\n";
        recommendation += "- Instruct patient to isolate immediately if symptoms develop\n";
        recommendation += "- Consider vaccination for future protection if not contraindicated\n";
      }
    }
   
    // Additional recommendations
    recommendation += "\n### Additional Recommendations:\n\n";
   
    if (data.highriskSetting === "Yes") {
      recommendation += "**High-risk setting considerations:**\n";
      recommendation += "- Exclude susceptible exposed persons from high-risk settings until 21 days after exposure\n";
      recommendation += "- If IG was administered, exclude until 28 days after exposure\n";
      recommendation += "- Consider facility-wide vaccination efforts if in a healthcare or institutional setting\n\n";
    }
   
    recommendation += "**Monitoring and follow-up:**\n";
    recommendation += "- Monitor for symptoms for 21 days after exposure\n";
    recommendation += "- Early symptoms: fever, malaise, cough, coryza, conjunctivitis, Koplik spots\n";
    recommendation += "- If symptoms develop, isolate immediately and confirm diagnosis\n";
    recommendation += "- Report suspected cases to public health authorities\n\n";
   
    recommendation += "**Note:** This guidance is based on CDC and IDSA recommendations. Clinical judgment should be used in individual cases.";
   
    return recommendation;
  };
 
  // Function to format age display for recommendation
  const formatAge = (age) => {
    // Return the original selection rather than a calculated value
    if (age === 0.25) return "<6 months";
    if (age === 0.75) return "6-11 months";
    if (age === 2.5) return "1-4 years";
    if (age === 12) return "5-18 years";
    if (age === 40) return "19-60 years";
    if (age === 65) return "61+ years";
    return age; // Fallback
  };

  // Function to restart the assessment
  const handleRestart = () => {
    setState({
      step: 0,
      chatHistory: [
        {
          sender: 'bot',
          message: "Welcome to the Measles Exposure Management Assistant. I'll help guide management decisions for patients with measles exposure."
        },
        {
          sender: 'bot',
          message: questions[0].text
        }
      ],
      userInput: '',
      patientData: {
        age: null,
        vaccineStatus: null,
        immunityStatus: null,
        pregnant: null,
        immunocompromised: null,
        exposureTime: null,
        highriskSetting: null
      },
      assessmentComplete: false
    });
  };

  // Initialize with first question
  React.useEffect(() => {
    if (state.chatHistory.length === 1) {
      setState(prevState => ({
        ...prevState,
        chatHistory: [
          ...prevState.chatHistory,
          {
            sender: 'bot',
            message: questions[0].text
          }
        ]
      }));
    }
  }, []);

  return (
    <div className="flex flex-col h-screen max-w-4xl mx-auto p-4 bg-gray-50">
      <header className="bg-blue-600 text-white p-4 rounded-t-lg shadow">
        <h1 className="text-xl font-bold">Measles Exposure Management Assistant</h1>
        <p className="text-sm">Evidence-based guidance for physicians managing measles exposures</p>
      </header>
     
      <div className="flex-1 overflow-auto p-4 bg-white rounded-lg shadow my-2">
        {state.chatHistory.map((chat, index) => (
          <div key={index} className={`mb-4 ${chat.sender === 'bot' ? 'pr-8' : 'pl-8'}`}>
            <div className={`p-3 rounded-lg ${
              chat.sender === 'bot'
                ? 'bg-blue-100 text-blue-900'
                : 'bg-gray-200 text-gray-900 ml-auto'
            }`}>
              {chat.sender === 'bot' ? (
                <div dangerouslySetInnerHTML={{
                  __html: chat.message.replace(/\*\*(.*?)\*\*/g, '<strong>$1</strong>')
                                      .replace(/\n/g, '<br />')
                }}/>
              ) : (
                chat.message
              )}
            </div>
          </div>
        ))}
      </div>
     
      <div className="mt-auto p-4 bg-white rounded-b-lg shadow">
        {!state.assessmentComplete ? (
          <div className="flex gap-2">
            <select
              value={state.userInput}
              onChange={(e) => setState(prevState => ({...prevState, userInput: e.target.value}))}
              className="flex-1 p-2 border border-gray-300 rounded"
            >
              <option value="">Select an option...</option>
              {state.step < questions.length && questions[state.step].options.map((option, index) => (
                <option key={index} value={option}>{option}</option>
              ))}
            </select>
            <button
              onClick={handleSubmit}
              disabled={!state.userInput}
              className={`px-4 py-2 rounded ${
                state.userInput 
                  ? 'bg-blue-600 text-white hover:bg-blue-700' 
                  : 'bg-gray-300 text-gray-500 cursor-not-allowed'
              }`}
            >
              Send
            </button>
          </div>
        ) : (
          <button
            onClick={handleRestart}
            className="w-full bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700"
          >
            Start New Assessment
          </button>
        )}
       
        <div className="mt-2 text-xs text-gray-500">
          <p>Based on CDC and IDSA guidelines for measles post-exposure management. For clinical guidance only.</p>
        </div>
      </div>
    </div>
  );
};

export default MeaslesExposureBot;
