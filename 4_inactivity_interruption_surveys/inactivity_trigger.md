# GTM Inactivity Trigger Survey Implementation Guide

This guide explains how to implement inactivity triggered surveys using Google Tag Manager (GTM) to gather feedback from users who may be experiencing confusion or having trouble navigating your website.

---

## Step 1: Define Inactivity Parameters

Before implementation, determine what constitutes meaningful inactivity on your site:

1. Common inactivity thresholds:
   - 30-45 seconds without mouse movement, scrolling, or clicking
   - 60+ seconds on forms without typing or selection
   - 2+ minutes on product pages without interaction

2. Consider which pages would benefit most from inactivity detection:
   - Checkout or conversion pages
   - Complex product configuration tools
   - Multi-step forms
   - Information-dense pages that might cause confusion

---

## Step 2: Create Inactivity Detector Tag

1. Go to the **Tags** section in your GTM workspace.
2. Click the **New** button.
3. Name the tag: **Inactivity Detector**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Only set up once per page
       if (window.inactivityDetectorInitialized) return;
       window.inactivityDetectorInitialized = true;
       
       // Check if survey already shown
       var cookieName = 'inactivitySurveyShown';
       if (document.cookie.indexOf(cookieName + '=') !== -1) return;
       
       // Configuration
       var config = {
         inactivityThreshold: 45000,  // 45 seconds of inactivity
         resetOnActivity: true,        // Reset timer on activity
         pageType: {{Page Type}} || 'general'
       };
       
       // Adjust threshold based on page type
       if (config.pageType === 'checkout' || config.pageType === 'form') {
         config.inactivityThreshold = 60000; // 60 seconds for forms/checkout
       } else if (config.pageType === 'product') {
         config.inactivityThreshold = 45000; // 45 seconds for product pages
       } else if (config.pageType === 'landing') {
         config.inactivityThreshold = 30000; // 30 seconds for landing pages
       }
       
       // Variables to track state
       var inactivityTimer;
       var surveyTriggered = false;
       
       // Function to reset the inactivity timer
       function resetInactivityTimer() {
         // Clear existing timer
         if (inactivityTimer) {
           clearTimeout(inactivityTimer);
         }
         
         // Don't set a new timer if survey already triggered
         if (surveyTriggered) return;
         
         // Set new timer
         inactivityTimer = setTimeout(function() {
           handleInactivity();
         }, config.inactivityThreshold);
       }
       
       // Function to handle inactivity
       function handleInactivity() {
         // Prevent multiple triggers
         if (surveyTriggered) return;
         surveyTriggered = true;
         
         // Log page state
         var pageState = {
           url: window.location.href,
           title: document.title,
           referrer: document.referrer,
           pageType: config.pageType,
           visibleElements: getVisibleElements(),
           scrollPosition: getScrollPosition(),
           formFields: getFormFields()
         };
         
         // Store page state for use in the survey
         window.inactivityPageState = pageState;
         
         // Trigger the survey
         dataLayer.push({
           'event': 'userInactivityDetected',
           'inactivityDuration': config.inactivityThreshold / 1000,
           'pageType': config.pageType,
           'pageTitle': document.title,
           'scrollPosition': pageState.scrollPosition.percent
         });
         
         // Set cookie to prevent showing again too soon
         document.cookie = cookieName + "=true; path=/; max-age=3600"; // 1 hour
       }
       
       // Helper function to get scroll position
       function getScrollPosition() {
         var scrollTop = window.pageYOffset || document.documentElement.scrollTop;
         var documentHeight = Math.max(
           document.body.scrollHeight, 
           document.body.offsetHeight,
           document.documentElement.clientHeight,
           document.documentElement.scrollHeight, 
           document.documentElement.offsetHeight
         );
         var windowHeight = window.innerHeight;
         var scrollPercent = (scrollTop / (documentHeight - windowHeight)) * 100;
         
         return {
           pixels: scrollTop,
           percent: Math.round(scrollPercent)
         };
       }
       
       // Helper function to detect visible elements that might be relevant
       function getVisibleElements() {
         var relevantElements = [];
         
         // Check for visible forms
         document.querySelectorAll('form').forEach(function(form) {
           if (isElementInViewport(form)) {
             relevantElements.push({
               type: 'form',
               id: form.id || 'unnamed-form'
             });
           }
         });
         
         // Check for visible CTAs
         document.querySelectorAll('button, a.btn, a.cta, .call-to-action').forEach(function(cta) {
           if (isElementInViewport(cta)) {
             relevantElements.push({
               type: 'cta',
               text: cta.innerText || 'unnamed-cta'
             });
           }
         });
         
         return relevantElements;
       }
       
       // Helper function to check if element is in viewport
       function isElementInViewport(el) {
         var rect = el.getBoundingClientRect();
         return (
           rect.top >= 0 &&
           rect.left >= 0 &&
           rect.bottom <= (window.innerHeight || document.documentElement.clientHeight) &&
           rect.right <= (window.innerWidth || document.documentElement.clientWidth)
         );
       }
       
       // Helper function to get form fields that might be causing issues
       function getFormFields() {
         var forms = [];
         
         document.querySelectorAll('form').forEach(function(form) {
           var fields = [];
           form.querySelectorAll('input, select, textarea').forEach(function(field) {
             fields.push({
               type: field.type || field.tagName.toLowerCase(),
               id: field.id || 'unnamed-field',
               name: field.name || '',
               hasValue: field.value.length > 0,
               isFocused: document.activeElement === field
             });
           });
           
           forms.push({
             id: form.id || 'unnamed-form',
             fields: fields
           });
         });
         
         return forms;
       }
       
       // Set up event listeners for user activity
       var activityEvents = ['mousedown', 'mousemove', 'keypress', 'scroll', 'click', 'touchstart', 'touchmove'];
       
       activityEvents.forEach(function(eventType) {
         document.addEventListener(eventType, function() {
           if (config.resetOnActivity) {
             resetInactivityTimer();
           }
         }, true);
       });
       
       // Initialize the inactivity timer
       resetInactivityTimer();
     })();
   </script>
   ```

7. Under **Triggering**, select **Window Loaded** trigger.
8. Click **Save**.

---

## Step 3: Create Page Type Variable

1. Go to **Variables** in your GTM workspace.
2. Click the **New** button under "User-Defined Variables".
3. Name the variable: **Page Type**.
4. Click **Variable Configuration**.
5. Choose **Custom JavaScript** as the variable type.
6. Enter the following code (customize based on your site's structure):

   ```javascript
   function() {
     var path = window.location.pathname;
     
     // Check URL patterns to determine page type
     if (path.match(/\/checkout\b/) || path.indexOf('/cart') > -1) {
       return 'checkout';
     } else if (path.match(/\/product\//) || path.match(/\/item\//)) {
       return 'product';
     } else if (document.querySelector('form[id*="checkout"], form[id*="payment"], form.checkout')) {
       return 'checkout';
     } else if (document.querySelector('form') && document.querySelectorAll('form input, form select').length > 3) {
       return 'form';
     } else if (document.referrer.indexOf('google.com') > -1 || document.referrer.indexOf('facebook.com') > -1) {
       return 'landing';
     }
     
     // Default
     return 'general';
   }
   ```

7. Click **Save**.

---

## Step 4: Create Inactivity Survey Display Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Inactivity Survey Display**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Get inactivity information from dataLayer
       var pageType = {{dlv - pageType}} || 'page';
       var pageTitle = {{dlv - pageTitle}} || document.title;
       var inactivityDuration = {{dlv - inactivityDuration}} || 45;
       var scrollPosition = {{dlv - scrollPosition}} || 0;
       
       // Create helper function
       function createSurvey() {
         // Create survey container
         var surveyContainer = document.createElement('div');
         surveyContainer.className = 'inactivity-help-survey';
         surveyContainer.style.cssText = 'position:fixed; top:50%; left:50%; transform:translate(-50%, -50%); width:400px; background:#fff; border-radius:10px; box-shadow:0 0 25px rgba(0,0,0,0.3); padding:25px; z-index:999999; font-family:Arial,sans-serif;';
         
         // Create background overlay
         var overlay = document.createElement('div');
         overlay.style.cssText = 'position:fixed; top:0; left:0; width:100%; height:100%; background:rgba(0,0,0,0.5); z-index:999998;';
         
         // Customize survey based on page type
         var helpText = "It looks like you've been inactive for a while. Can we help you with anything?";
         var surveyTitle = "Need Help?";
         
         if (pageType === 'checkout') {
           helpText = "Are you having trouble completing your purchase?";
           surveyTitle = "Need Help with Your Order?";
         } else if (pageType === 'form') {
           helpText = "Do you need assistance completing this form?";
           surveyTitle = "Need Help with This Form?";
         } else if (pageType === 'product') {
           helpText = "Do you have questions about this product?";
           surveyTitle = "Questions About This Product?";
         }
         
         // Survey content
         surveyContainer.innerHTML = `
           <div style="text-align:center; margin-bottom:20px;">
             <h2 style="margin:0 0 10px; color:#333; font-size:22px;">${surveyTitle}</h2>
           </div>
           <p style="margin:0 0 20px; color:#444; font-size:16px; text-align:center;">${helpText}</p>
           <div style="display:flex; flex-direction:column; gap:10px; margin-bottom:20px;">
             <button class="inactivity-option" data-value="yes-help" style="padding:12px; background:#f5f5f5; border:none; border-radius:5px; cursor:pointer; font-size:15px; text-align:left;">
               Yes, I need help
             </button>
             <button class="inactivity-option" data-value="just-browsing" style="padding:12px; background:#f5f5f5; border:none; border-radius:5px; cursor:pointer; font-size:15px; text-align:left;">
               No thanks, just browsing
             </button>
             <button class="inactivity-option" data-value="found-issue" style="padding:12px; background:#f5f5f5; border:none; border-radius:5px; cursor:pointer; font-size:15px; text-align:left;">
               I found something confusing or broken
             </button>
           </div>
           <div id="helpFeedback" style="display:none; margin-bottom:20px;">
             <textarea id="helpFeedbackText" placeholder="Please tell us what you need help with..." style="width:100%; padding:12px; border:1px solid #ddd; border-radius:5px; resize:vertical; min-height:80px; box-sizing:border-box; font-family:inherit;"></textarea>
             <button id="submitHelpSurvey" style="width:100%; margin-top:15px; padding:12px; background:#4a90e2; color:#fff; border:none; border-radius:5px; cursor:pointer; font-weight:bold; font-size:16px;">Submit</button>
           </div>
           <div style="text-align:center;">
             <button id="closeHelpSurvey" style="background:none; border:none; color:#999; font-size:14px; cursor:pointer; text-decoration:underline;">Close</button>
           </div>
         `;
         
         // Add to page
         document.body.appendChild(overlay);
         document.body.appendChild(surveyContainer);
         
         // Prevent page scrolling when survey is open
         document.body.style.overflow = 'hidden';
         
         // Close survey function
         function closeSurvey() {
           document.body.removeChild(surveyContainer);
           document.body.removeChild(overlay);
           document.body.style.overflow = '';
         }
         
         // Close button functionality
         document.getElementById('closeHelpSurvey').addEventListener('click', closeSurvey);
         
         // Option button functionality
         var optionButtons = document.querySelectorAll('.inactivity-option');
         var selectedOption = null;
         
         optionButtons.forEach(function(button) {
           button.addEventListener('click', function() {
             // Get the option value
             selectedOption = this.getAttribute('data-value');
             
             // Reset all buttons
             optionButtons.forEach(function(btn) {
               btn.style.background = '#f5f5f5';
               btn.style.fontWeight = 'normal';
             });
             
             // Highlight selected button
             this.style.background = '#e0e7ff';
             this.style.fontWeight = 'bold';
             
             // Show feedback text area for help or issue options
             if (selectedOption === 'yes-help' || selectedOption === 'found-issue') {
               document.getElementById('helpFeedback').style.display = 'block';
             } else {
               // For "just browsing", submit immediately
               submitFeedback();
             }
           });
         });
         
         // Submit button functionality
         var submitButton = document.getElementById('submitHelpSurvey');
         if (submitButton) {
           submitButton.addEventListener('click', function() {
             submitFeedback();
           });
         }
         
         // Function to submit feedback
         function submitFeedback() {
           if (!selectedOption) return;
           
           var feedbackText = '';
           var feedbackTextArea = document.getElementById('helpFeedbackText');
           if (feedbackTextArea && feedbackTextArea.value) {
             feedbackText = feedbackTextArea.value;
           }
           
           // Send the data
           dataLayer.push({
             'event': 'inactivitySurveyResponse',
             'pageType': pageType,
             'pageTitle': pageTitle,
             'inactivityDuration': inactivityDuration,
             'surveyResponse': selectedOption,
             'surveyFeedback': feedbackText,
             'scrollPosition': scrollPosition
           });
           
           // For help requests, show a thank you message with additional contact options
           if (selectedOption === 'yes-help') {
             surveyContainer.innerHTML = `
               <div style="text-align:center; padding:20px;">
                 <h3 style="margin-bottom:15px; color:#333;">We're Here to Help!</h3>
                 <p style="color:#666; margin-bottom:20px;">Our team is ready to assist you. Choose an option below:</p>
                 <div style="display:flex; flex-direction:column; gap:10px;">
                   <a href="/contact" style="padding:10px; background:#4a90e2; color:#fff; text-decoration:none; border-radius:5px; font-weight:bold;">Contact Support</a>
                   <a href="#" onclick="window.Intercom && window.Intercom('show'); return false;" style="padding:10px; background:#f5f5f5; color:#333; text-decoration:none; border-radius:5px;">Chat with Us</a>
                   <button onclick="window.open('tel:+1800123456')" style="padding:10px; background:#f5f5f5; border:none; border-radius:5px; cursor:pointer;">Call Us: 1-800-123-456</button>
                 </div>
               </div>
             `;
           } else {
             // For other responses, show a simple thank you
             surveyContainer.innerHTML = '<div style="text-align:center; padding:20px;"><h3 style="margin-bottom:15px; color:#333;">Thank You!</h3><p style="color:#666;">We appreciate your feedback.</p></div>';
             
             // Remove after delay
             setTimeout(function() {
               closeSurvey();
             }, 3000);
           }
         }
       }
       
       // Display the survey after a small delay
       setTimeout(createSurvey, 500);
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Inactivity Survey Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `userInactivityDetected`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your survey tag, select **Inactivity Survey Trigger**.
9. Click **Save**.

---

## Step 5: Create Data Collection Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Inactivity Survey Response Collector**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code (replace with your actual survey endpoint):

   ```html
   <script>
     (function() {
       // Get response data from dataLayer
       var dlEvent = {{dataLayer}};
       
       // Only proceed if this is an inactivity survey response
       if (dlEvent.event !== 'inactivitySurveyResponse') return;
       
       // Prepare data for sending
       var responseData = {
         type: 'inactivitySurvey',
         pageType: dlEvent.pageType,
         pageTitle: dlEvent.pageTitle,
         inactivityDuration: dlEvent.inactivityDuration,
         response: dlEvent.surveyResponse,
         feedback: dlEvent.surveyFeedback,
         scrollPosition: dlEvent.scrollPosition,
         url: window.location.href,
         timestamp: new Date().toISOString(),
         userId: {{User ID}} || 'anonymous'
       };
       
       // Send to your survey endpoint
       fetch('https://your-survey-endpoint.com/api/inactivity-survey', {
         method: 'POST',
         headers: {
           'Content-Type': 'application/json',
         },
         body: JSON.stringify(responseData)
       })
       .then(response => response.json())
       .then(data => console.log('Inactivity survey submitted:', data))
       .catch(error => console.error('Error submitting inactivity survey:', error));
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Inactivity Survey Response Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `inactivitySurveyResponse`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your data collection tag, select **Inactivity Survey Response Trigger**.
9. Click **Save**.

---

## Step 6: Variable Setup for DataLayer Values

1. Go to **Variables** in your GTM workspace.
2. Click the **New** button under "User-Defined Variables".
3. Create the following variables:

   a. **dlv - pageType**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `pageType`

   b. **dlv - pageTitle**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `pageTitle`

   c. **dlv - inactivityDuration**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `inactivityDuration`
   
   d. **dlv - scrollPosition**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `scrollPosition`

4. Click **Save** for each variable.

---

## Step 7: Create Targeting Conditions (Optional)

To make your inactivity detection more effective, you can add conditions to only trigger on certain pages:

1. Go to **Triggers** in your GTM workspace.
2. Edit your **Inactivity Survey Trigger**.
3. Under the "This trigger fires on" section, select **Some Custom Events**.
4. Add the following conditions (customize as needed):
   - **Page Path** contains `/checkout/` OR
   - **Page Path** contains `/product/` OR
   - **Page Type** equals `form`
5. Click **Save**.

---

## Step 8: Test and Publish

1. Click the **Preview** button in GTM.
2. Visit one of your target pages.
3. Remain inactive for the duration of your inactivity threshold.
4. Verify that the survey appears.
5. Test each response option.
6. Verify that the "I need help" option shows contact methods.
7. Verify data is being collected properly.
8. If everything functions correctly, publish your changes.

---

## Optional Enhancements

- **Custom Logic for Different Pages:** Create various inactivity detection rules for different parts of your site.
- **AI-Based Help Suggestions:** Analyze what the user has been doing and offer specific help based on detected issues.
- **Live Chat Integration:** Directly connect users who need help to your customer service team via live chat.
- **Focus on Common Sticking Points:** Analyze your analytics data to identify pages with high bounce rates or abandonment, and focus inactivity detection there.
- **Progressive Assistance:** Start with simple help options, then escalate to more interactive assistance if the user continues to be inactive.

---

## Advanced: Form Field Analysis

For more detailed insights on form pages, you can enhance the inactivity detector to analyze which form fields might be causing confusion:

```javascript
// Enhanced form field analysis
function analyzeFormFields() {
  var forms = document.querySelectorAll('form');
  var formIssues = [];
  
  forms.forEach(function(form) {
    var formData = {
      id: form.id || form.getAttribute('name') || 'unnamed-form',
      fields: [],
      potentialIssues: []
    };
    
    // Check each field
    var fields = form.querySelectorAll('input, select, textarea');
    fields.forEach(function(field, index) {
      var fieldData = {
        type: field.type || field.tagName.toLowerCase(),
        id: field.id || field.name || 'field-' + index,
        value: field.value,
        focused: document.activeElement === field,
        visible: isElementInViewport(field),
        hasError: field.classList.contains('error') || field.classList.contains('invalid') || field.parentElement.classList.contains('error')
      };
      
      formData.fields.push(fieldData);
      
      // Detect potential issues
      if (fieldData.hasError) {
        formData.potentialIssues.push({
          type: 'error',
          field: fieldData.id,
          message: 'Field has error state'
        });
      }
      
      if (fieldData.focused && !fieldData.value && fieldData.visible) {
        formData.potentialIssues.push({
          type: 'empty_focused',
          field: fieldData.id,
          message: 'User is stuck on empty field'
        });
      }
    });
    
    // Check if form is almost complete but abandoned
    var filledFields = formData.fields.filter(function(f) { return f.value.length > 0; }).length;
    var completionRate = filledFields / formData.fields.length;
    
    if (completionRate > 0.7 && completionRate < 1) {
      formData.potentialIssues.push({
        type: 'almost_complete',
        completionRate: completionRate,
        message: 'Form is ' + Math.round(completionRate * 100) + '% complete but abandoned'
      });
    }
    
    formIssues.push(formData);
  });
  
  return formIssues;
}
```

This enhanced analysis helps identify specific form fields that might be causing users to become inactive, allowing you to provide more targeted assistance.

--- 