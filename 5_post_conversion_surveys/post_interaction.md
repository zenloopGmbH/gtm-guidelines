# GTM Post-Interaction Survey Implementation Guide

This guide explains how to implement post-interaction surveys using Google Tag Manager (GTM) to gather feedback immediately after a user completes a significant interaction with your website, such as a customer support chat, product demo, or form submission.

---

## Step 1: Identify Key Interactions

Before implementation, determine which user interactions are significant enough to warrant feedback:

1. Common user interactions to track:
   - Live chat sessions
   - Product demo completions
   - Content downloads
   - Video views
   - Calculator or tool usage
   - Appointment scheduling
   - Multi-step form submissions

2. Parameters to track for each interaction:
   - Interaction type
   - Duration
   - Specific content/product involved
   - User segment
   - Outcome (successful or unsuccessful)

---

## Step 2: Set Up Custom Event Tracking

First, you need to ensure your interactions are being tracked through dataLayer events:

### For Chat Interactions

If you're using a chat service, check if they provide event tracking. If not, implement your own:

```javascript
// Example for tracking chat end
function trackChatEnd(chatData) {
  dataLayer.push({
    'event': 'chatEnded',
    'chatAgent': chatData.agentName,
    'chatDuration': chatData.duration,
    'chatTopic': chatData.topic
  });
}
```

### For Product Demos

```javascript
// Example for tracking demo completion
document.querySelector('.demo-complete-button').addEventListener('click', function() {
  dataLayer.push({
    'event': 'demoCompleted',
    'demoProduct': 'Product Name',
    'demoLength': 5 // minutes
  });
});
```

### For Video Views

```javascript
// Example for tracking video completion
videoPlayer.on('ended', function() {
  dataLayer.push({
    'event': 'videoCompleted',
    'videoTitle': 'Tutorial Video',
    'videoDuration': 120 // seconds
  });
});
```

---

## Step 3: Create Interaction Event Trigger

1. Navigate to **Triggers** in your GTM workspace.
2. Click the **New** button.
3. Name the trigger: **Significant Interaction Trigger**.
4. Click **Trigger Configuration**.
5. Select **Custom Event**.
6. For **Event Name**, enter the events you want to track, such as:
   - `chatEnded`
   - `demoCompleted`
   - `videoCompleted`
   - (You can create multiple triggers for different events)
7. Click **Save**.

---

## Step 4: Create Interaction Survey Delay Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Post-Interaction Survey Delay**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Get interaction details from dataLayer
       var dlEvent = {{dataLayer}};
       var eventName = dlEvent.event;
       
       // Only proceed for target events
       var targetEvents = ['chatEnded', 'demoCompleted', 'videoCompleted', 'toolUsed'];
       if (!targetEvents.includes(eventName)) return;
       
       // Prevent multiple triggers
       if (window.interactionSurveyTriggered) return;
       window.interactionSurveyTriggered = true;
       
       // Determine interaction type and details
       var interactionType = getInteractionType(eventName);
       var interactionDetails = getInteractionDetails(dlEvent);
       
       // Store for later use
       window.interactionData = {
         type: interactionType,
         details: interactionDetails,
         timestamp: new Date().toISOString()
       };
       
       // Check if survey already shown recently
       var cookieName = 'interactionSurvey_' + interactionType;
       if (document.cookie.indexOf(cookieName + '=') !== -1) return;
       
       // Determine appropriate delay
       var delayTime = 1000; // 1 second default
       
       // Different delays for different interactions
       if (interactionType === 'chat') {
         delayTime = 1500; // 1.5 seconds
       } else if (interactionType === 'demo') {
         delayTime = 2000; // 2 seconds
       } else if (interactionType === 'video') {
         delayTime = 500; // 0.5 seconds
       }
       
       // Trigger survey after delay
       setTimeout(function() {
         dataLayer.push({
           'event': 'showInteractionSurvey',
           'interactionType': interactionType,
           'interactionDetails': JSON.stringify(interactionDetails)
         });
         
         // Set cookie to avoid showing too frequently
         document.cookie = cookieName + "=true; path=/; max-age=86400"; // 24 hours
       }, delayTime);
       
       // Helper function to determine interaction type
       function getInteractionType(eventName) {
         if (eventName === 'chatEnded') return 'chat';
         if (eventName === 'demoCompleted') return 'demo';
         if (eventName === 'videoCompleted') return 'video';
         if (eventName === 'toolUsed') return 'tool';
         return 'general';
       }
       
       // Helper function to extract interaction details
       function getInteractionDetails(event) {
         var details = {};
         
         // For chat interactions
         if (event.chatAgent) {
           details = {
             agent: event.chatAgent,
             duration: event.chatDuration || 0,
             topic: event.chatTopic || 'general'
           };
         }
         
         // For demo interactions
         else if (event.demoProduct) {
           details = {
             product: event.demoProduct,
             length: event.demoLength || 0
           };
         }
         
         // For video interactions
         else if (event.videoTitle) {
           details = {
             title: event.videoTitle,
             duration: event.videoDuration || 0
           };
         }
         
         // For tool interactions
         else if (event.toolId) {
           details = {
             id: event.toolId,
             result: event.toolResult
           };
         }
         
         return details;
       }
     })();
   </script>
   ```

7. Under **Triggering**, select your **Significant Interaction Trigger**.
8. Click **Save**.

---

## Step 5: Create Survey Display Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Post-Interaction Survey Display**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Get interaction details from dataLayer
       var interactionType = {{dlv - interactionType}} || 'general';
       var interactionDetailsString = {{dlv - interactionDetails}} || '{}';
       var interactionDetails = {};
       
       try {
         interactionDetails = JSON.parse(interactionDetailsString);
       } catch (e) {
         console.error('Error parsing interaction details:', e);
       }
       
       // Create survey container
       var surveyContainer = document.createElement('div');
       surveyContainer.className = 'interaction-feedback-survey';
       surveyContainer.style.cssText = 'position:fixed; bottom:20px; right:20px; width:380px; background:#fff; border-radius:10px; box-shadow:0 0 20px rgba(0,0,0,0.25); padding:20px; z-index:999999; font-family:Arial,sans-serif; transition:transform 0.3s ease-in-out; transform:translateY(20px);';
       
       // Customize survey based on interaction type
       var surveyTitle = "Quick Feedback";
       var surveyTitleDetail = "";
       var surveyQuestion = "How would you rate your experience?";
       var surveyOptions = [];
       
       if (interactionType === 'chat') {
         surveyTitle = "Chat Feedback";
         surveyTitleDetail = interactionDetails.agent ? `with ${interactionDetails.agent}` : '';
         surveyQuestion = "How helpful was your chat conversation?";
         surveyOptions = [
           "Very helpful",
           "Somewhat helpful",
           "Not helpful"
         ];
       } else if (interactionType === 'demo') {
         surveyTitle = "Demo Feedback";
         surveyTitleDetail = interactionDetails.product ? `for ${interactionDetails.product}` : '';
         surveyQuestion = "How useful was this product demonstration?";
         surveyOptions = [
           "Very useful",
           "Somewhat useful",
           "Not useful"
         ];
       } else if (interactionType === 'video') {
         surveyTitle = "Video Feedback";
         surveyTitleDetail = interactionDetails.title ? `for "${interactionDetails.title}"` : '';
         surveyQuestion = "How would you rate this video?";
         surveyOptions = [
           "Very helpful",
           "Somewhat helpful",
           "Not helpful"
         ];
       } else if (interactionType === 'tool') {
         surveyTitle = "Tool Feedback";
         surveyQuestion = "How would you rate this tool?";
         surveyOptions = [
           "Very useful",
           "Somewhat useful",
           "Not useful"
         ];
       }
       
       // Generate options HTML
       var optionsHTML = '';
       surveyOptions.forEach(function(option, index) {
         optionsHTML += `
           <button class="survey-option" data-value="${index + 1}" style="width:100%; padding:10px; margin-bottom:8px; background:#f5f5f5; border:none; border-radius:5px; cursor:pointer; text-align:left; font-size:14px;">
             ${option}
           </button>
         `;
       });
       
       // Survey content
       surveyContainer.innerHTML = `
         <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:15px;">
           <div>
             <h3 style="margin:0; color:#333; font-size:18px;">${surveyTitle} ${surveyTitleDetail}</h3>
           </div>
           <button id="closeInteractionSurvey" style="background:none; border:none; cursor:pointer; font-size:20px; color:#999;">&times;</button>
         </div>
         <p style="margin:0 0 15px; color:#444; font-size:15px;">${surveyQuestion}</p>
         <div style="margin-bottom:15px;">
           ${optionsHTML}
         </div>
         <div id="additionalFeedback" style="display:none; margin-bottom:15px;">
           <p style="margin:0 0 5px; color:#444; font-size:14px;">Would you like to share any additional comments?</p>
           <textarea id="feedbackText" placeholder="Your comments..." style="width:100%; padding:10px; border:1px solid #ddd; border-radius:5px; resize:vertical; min-height:70px; box-sizing:border-box; font-family:inherit;"></textarea>
           <button id="submitInteractionSurvey" style="width:100%; margin-top:10px; padding:10px; background:#4a90e2; color:#fff; border:none; border-radius:5px; cursor:pointer; font-weight:bold;">Submit Feedback</button>
         </div>
       `;
       
       // Add to page
       document.body.appendChild(surveyContainer);
       
       // Animate in
       setTimeout(function() {
         surveyContainer.style.transform = 'translateY(0)';
       }, 10);
       
       // Store selected rating
       var selectedRating = null;
       
       // Close button functionality
       document.getElementById('closeInteractionSurvey').addEventListener('click', function() {
         surveyContainer.style.transform = 'translateY(20px)';
         setTimeout(function() {
           if (document.body.contains(surveyContainer)) {
             document.body.removeChild(surveyContainer);
           }
         }, 300);
       });
       
       // Option button functionality
       var optionButtons = document.querySelectorAll('.survey-option');
       optionButtons.forEach(function(button) {
         button.addEventListener('click', function() {
           // Get selected value
           selectedRating = this.getAttribute('data-value');
           
           // Reset all buttons
           optionButtons.forEach(function(btn) {
             btn.style.background = '#f5f5f5';
             btn.style.fontWeight = 'normal';
           });
           
           // Highlight selected button
           this.style.background = '#e0e7ff';
           this.style.fontWeight = 'bold';
           
           // Show additional feedback section
           document.getElementById('additionalFeedback').style.display = 'block';
           
           // If negative rating, scroll the survey to show the text area
           if (parseInt(selectedRating) === 3) { // "Not helpful/useful" option
             surveyContainer.scrollTop = surveyContainer.scrollHeight;
           }
         });
       });
       
       // Submit button functionality
       document.getElementById('submitInteractionSurvey').addEventListener('click', function() {
         if (!selectedRating) return;
         
         var feedbackText = document.getElementById('feedbackText').value || '';
         
         // Send the data
         dataLayer.push({
           'event': 'interactionSurveyResponse',
           'interactionType': interactionType,
           'interactionRating': selectedRating,
           'interactionFeedback': feedbackText,
           'interactionDetails': interactionDetailsString
         });
         
         // Show thank you message
         surveyContainer.innerHTML = `
           <div style="text-align:center; padding:20px;">
             <h3 style="margin-bottom:10px; color:#333;">Thank You!</h3>
             <p style="color:#666; margin-bottom:15px;">Your feedback helps us improve our service.</p>
           </div>
         `;
         
         // Remove after delay
         setTimeout(function() {
           surveyContainer.style.transform = 'translateY(20px)';
           setTimeout(function() {
             if (document.body.contains(surveyContainer)) {
               document.body.removeChild(surveyContainer);
             }
           }, 300);
         }, 2000);
       });
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Show Interaction Survey Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `showInteractionSurvey`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your survey tag, select **Show Interaction Survey Trigger**.
9. Click **Save**.

---

## Step 6: Create Data Collection Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Interaction Survey Response Collector**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code (replace with your actual survey endpoint):

   ```html
   <script>
     (function() {
       // Get response data from dataLayer
       var dlEvent = {{dataLayer}};
       
       // Only proceed if this is an interaction survey response
       if (dlEvent.event !== 'interactionSurveyResponse') return;
       
       // Prepare data for sending
       var responseData = {
         type: 'interactionSurvey',
         interactionType: dlEvent.interactionType,
         rating: dlEvent.interactionRating,
         feedback: dlEvent.interactionFeedback,
         details: dlEvent.interactionDetails,
         url: window.location.href,
         timestamp: new Date().toISOString(),
         userId: {{User ID}} || 'anonymous'
       };
       
       // Send to your survey endpoint
       fetch('https://your-survey-endpoint.com/api/interaction-survey', {
         method: 'POST',
         headers: {
           'Content-Type': 'application/json',
         },
         body: JSON.stringify(responseData)
       })
       .then(response => response.json())
       .then(data => console.log('Interaction survey submitted:', data))
       .catch(error => console.error('Error submitting interaction survey:', error));
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Interaction Survey Response Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `interactionSurveyResponse`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your data collection tag, select **Interaction Survey Response Trigger**.
9. Click **Save**.

---

## Step 7: Variable Setup for DataLayer Values

1. Go to **Variables** in your GTM workspace.
2. Click the **New** button under "User-Defined Variables".
3. Create the following variables:

   a. **dlv - interactionType**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `interactionType`

   b. **dlv - interactionDetails**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `interactionDetails`

4. Click **Save** for each variable.

---

## Step 8: Test and Publish

1. Click the **Preview** button in GTM.
2. Complete one of your tracked interactions (chat, demo, video, etc.).
3. Verify that after the configured delay, the survey appears.
4. Test selecting different options and submitting feedback.
5. Verify that the data is being collected properly.
6. If everything functions correctly, publish your changes.

---

## Implementation Examples for Common Platforms

### Intercom Chat Integration

```javascript
// Add this to your Intercom implementation
window.intercomSettings = {
  // Your existing settings...
  
  // This creates a custom event when chat ends
  onHide: function() {
    if (window.Intercom && Intercom.booted) {
      var chatData = {
        agentName: 'Intercom Agent', // Generic or try to get actual agent name
        duration: calculateChatDuration(),
        topic: getChatTopic() || 'General Support'
      };
      
      dataLayer.push({
        'event': 'chatEnded',
        'chatAgent': chatData.agentName,
        'chatDuration': chatData.duration,
        'chatTopic': chatData.topic
      });
    }
  }
};

function calculateChatDuration() {
  // Calculate time difference from chat start (store when chat opened)
  if (window.chatStartTime) {
    return Math.round((new Date() - window.chatStartTime) / 1000);
  }
  return 0;
}

function getChatTopic() {
  // Try to determine topic from conversation
  // This is highly dependent on your Intercom implementation
  return '';
}

// Add this for when chat starts
Intercom('onShow', function() {
  window.chatStartTime = new Date();
});
```

### Video Player Integration (Example for Wistia)

```html
<script>
window._wq = window._wq || [];
_wq.push({ id: "your_video_id", onEnd: function(video) {
  dataLayer.push({
    'event': 'videoCompleted',
    'videoTitle': video.name(),
    'videoDuration': video.duration()
  });
}});
</script>
```

### Product Demo Integration

```html
<!-- Add this to your product demo page -->
<script>
document.addEventListener('DOMContentLoaded', function() {
  var demoCompletionButtons = document.querySelectorAll('.demo-complete, .next-steps-button, .finish-demo');
  
  demoCompletionButtons.forEach(function(button) {
    button.addEventListener('click', function() {
      // Get product name from page
      var productName = document.querySelector('h1, .product-title').textContent;
      
      // Calculate demo duration if you stored start time
      var demoDuration = 0;
      if (window.demoStartTime) {
        demoDuration = Math.round((new Date() - window.demoStartTime) / 60000); // in minutes
      }
      
      dataLayer.push({
        'event': 'demoCompleted',
        'demoProduct': productName,
        'demoLength': demoDuration
      });
    });
  });
  
  // Store demo start time
  window.demoStartTime = new Date();
});
</script>
```

---

## Optional Enhancements

- **Contextual Questions:** Create different survey questions based on the specific interaction details.
  ```javascript
  // Example: Different questions for different chat topics
  if (interactionDetails.topic === 'technical-support') {
    surveyQuestion = "How well did we resolve your technical issue?";
  } else if (interactionDetails.topic === 'billing') {
    surveyQuestion = "How satisfied are you with the billing assistance you received?";
  }
  ```

- **Rating Scale Customization:** Adjust rating scales based on interaction type.
  ```javascript
  // Example: NPS scale for important interactions
  if (interactionType === 'demo' && isPotentialHighValueCustomer()) {
    // Use 0-10 NPS scale instead of 3-point scale
  }
  ```

- **Follow-up Integration:** Based on negative feedback, trigger follow-up actions.
  ```javascript
  // Example: Trigger customer support follow-up for negative ratings
  if (parseInt(selectedRating) === 3) { // "Not helpful" option
    // Store for customer support follow-up
    storeForFollowUp(interactionDetails, feedbackText);
  }
  
  function storeForFollowUp(details, feedback) {
    // Implementation depends on your CRM/support system
  }
  ```

- **Session Recording Integration:** Link survey responses to session recordings.
  ```javascript
  // If you use a session recording tool, include the recording ID
  if (window.sessionRecordingId) {
    responseData.recordingId = window.sessionRecordingId;
  }
  ```

---

## Advanced: Intelligent Survey Timing

For more sophisticated timing decisions, consider adding intelligent logic:

```javascript
function determineOptimalSurveyDelay(interactionType, interactionDetails) {
  // Base delay
  var delay = 1000;
  
  // Adjust based on interaction duration
  if (interactionDetails.duration) {
    // For longer interactions, give more time to process
    if (interactionDetails.duration > 300) { // > 5 minutes
      delay = 3000; // 3 seconds
    } else if (interactionDetails.duration < 60) { // < 1 minute
      delay = 500; // Half second
    }
  }
  
  // Adjust based on interaction outcome (if available)
  if (interactionDetails.outcome === 'successful') {
    // Show survey faster for successful interactions
    delay = Math.max(500, delay - 500);
  } else if (interactionDetails.outcome === 'unsuccessful') {
    // Give more time for unsuccessful interactions
    delay = delay + 1000;
  }
  
  // Consider page context
  if (isCheckoutPage() || isHighTrafficPage()) {
    // Shorter delay on important pages to catch user before they leave
    delay = Math.min(delay, 1000);
  }
  
  return delay;
}
```

By intelligently timing your surveys based on interaction context, you can significantly improve response rates and collect more relevant feedback.

--- 