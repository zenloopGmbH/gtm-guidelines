# GTM Button Click Survey Implementation Guide

This guide explains how to implement button click triggered surveys using Google Tag Manager (GTM) to gather feedback when users interact with specific buttons or call-to-action elements on your website.

---

## Step 1: Identify Target Buttons

First, identify the buttons or CTAs you want to track:

1. Determine which buttons are most important for user journeys:
   - "Add to Cart" buttons
   - "Sign Up" or "Register" buttons
   - "Download" buttons
   - "Contact Us" buttons
   - Navigation elements

2. Ensure these buttons have unique identifiers:
   - IDs
   - Classes
   - Data attributes
   - Unique text content
   - Clear CSS selectors

---

## Step 2: Create Button Click Trigger

1. Navigate to **Triggers** in your GTM workspace.
2. Click the **New** button.
3. Name the trigger: **CTA Button Click Trigger**.
4. Click **Trigger Configuration**.
5. Select **Click - All Elements**.
6. Choose **Some Clicks**.
7. Set up the conditions to identify your target buttons:
   - **Click ID** equals `addToCartButton` (for buttons with specific IDs)
   - OR **Click Classes** contains `signup-button` (for buttons with specific classes)
   - OR **Click Text** equals `Download Now` (for buttons with specific text)
   - OR **Click Element** matches CSS selector `.product-page .cta-button` (for buttons with specific CSS selectors)
8. Click **Save**.

---

## Step 3: Create Button Click Data Collection Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Button Click Data Collector**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Get the clicked element
       var clickElement = {{Click Element}};
       if (!clickElement) return;
       
       // Collect information about the clicked button
       var buttonData = {
         id: clickElement.id || 'no-id',
         text: clickElement.innerText || clickElement.value || 'no-text',
         url: window.location.href,
         path: window.location.pathname,
         pageTitle: document.title
       };
       
       // Store this info for use in the survey
       window.clickedButtonData = buttonData;
       
       // Check if survey already shown for this button type
       var buttonType = buttonData.id || buttonData.text;
       var cookieName = 'buttonSurvey_' + buttonType.replace(/\s+/g, '_').toLowerCase();
       
       if (document.cookie.indexOf(cookieName + '=') !== -1) return;
       
       // Set survey parameters based on the button
       var surveyParams = {
         buttonType: buttonType,
         surveyDelay: 500,  // Show quickly after button click
         cookieName: cookieName
       };
       
       // Wait briefly, then show the survey to allow the main action to proceed
       setTimeout(function() {
         dataLayer.push({
           'event': 'buttonSurveyTrigger',
           'buttonType': surveyParams.buttonType,
           'buttonText': buttonData.text,
           'buttonPath': buttonData.path
         });
         
         // Set cookie to prevent showing again too soon
         document.cookie = surveyParams.cookieName + "=true; path=/; max-age=86400"; // 24 hours
       }, surveyParams.surveyDelay);
     })();
   </script>
   ```

7. Under **Triggering**, select your **CTA Button Click Trigger**.
8. Click **Save**.

---

## Step 4: Create Button Survey Display Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Button Survey Display**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Get information about the button click from dataLayer
       var buttonType = {{dlv - buttonType}} || 'this button';
       var buttonText = {{dlv - buttonText}} || 'the button';
       
       // Create survey container
       var surveyContainer = document.createElement('div');
       surveyContainer.className = 'button-feedback-survey';
       surveyContainer.style.cssText = 'position:fixed; top:50%; left:50%; transform:translate(-50%, -50%); width:400px; background:#fff; border-radius:10px; box-shadow:0 0 20px rgba(0,0,0,0.3); padding:25px; z-index:999999; font-family:Arial,sans-serif;';
       
       // Customize question based on the button type
       var questionText = "Why did you click '" + buttonText + "'?";
       if (buttonType.includes('cart') || buttonType.includes('buy')) {
         questionText = "What made you decide to add this item to your cart?";
       } else if (buttonType.includes('signup') || buttonType.includes('register')) {
         questionText = "What features are you most interested in?";
       } else if (buttonType.includes('download')) {
         questionText = "What will you use this download for?";
       }
       
       // Survey content
       surveyContainer.innerHTML = `
         <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:20px;">
           <h3 style="margin:0; color:#333; font-size:20px;">Quick Question</h3>
           <button id="closeButtonSurvey" style="background:none; border:none; cursor:pointer; font-size:22px; color:#999;">&times;</button>
         </div>
         <p style="margin:0 0 20px; color:#666; font-size:16px;">${questionText}</p>
         <div id="buttonSurveyOptions" style="margin-bottom:20px;">
           <div class="survey-option" style="padding:10px; margin-bottom:8px; border:1px solid #ddd; border-radius:5px; cursor:pointer;">
             <input type="radio" name="buttonSurvey" id="option1" value="option1">
             <label for="option1" style="cursor:pointer; margin-left:8px;">I'm researching options</label>
           </div>
           <div class="survey-option" style="padding:10px; margin-bottom:8px; border:1px solid #ddd; border-radius:5px; cursor:pointer;">
             <input type="radio" name="buttonSurvey" id="option2" value="option2">
             <label for="option2" style="cursor:pointer; margin-left:8px;">I'm ready to proceed</label>
           </div>
           <div class="survey-option" style="padding:10px; margin-bottom:8px; border:1px solid #ddd; border-radius:5px; cursor:pointer;">
             <input type="radio" name="buttonSurvey" id="option3" value="option3">
             <label for="option3" style="cursor:pointer; margin-left:8px;">I'm just exploring</label>
           </div>
           <div class="survey-option" style="padding:10px; border:1px solid #ddd; border-radius:5px; cursor:pointer;">
             <input type="radio" name="buttonSurvey" id="option4" value="option4">
             <label for="option4" style="cursor:pointer; margin-left:8px;">Other (please specify)</label>
           </div>
         </div>
         <div id="otherReason" style="display:none; margin-bottom:20px;">
           <textarea id="otherReasonText" placeholder="Please specify..." style="width:100%; padding:10px; border:1px solid #ddd; border-radius:5px; resize:vertical; min-height:80px; box-sizing:border-box;"></textarea>
         </div>
         <button id="submitButtonSurvey" style="width:100%; padding:12px; background:#4a90e2; color:#fff; border:none; border-radius:5px; cursor:pointer; font-weight:bold; font-size:16px;">Submit</button>
       `;
       
       // Add to page
       document.body.appendChild(surveyContainer);
       
       // Add overlay
       var overlay = document.createElement('div');
       overlay.style.cssText = 'position:fixed; top:0; left:0; width:100%; height:100%; background:rgba(0,0,0,0.5); z-index:999998;';
       document.body.appendChild(overlay);
       
       // Make survey options clickable
       var surveyOptions = document.querySelectorAll('.survey-option');
       surveyOptions.forEach(function(option) {
         option.addEventListener('click', function() {
           // Find the radio input within this option
           var radio = this.querySelector('input[type="radio"]');
           radio.checked = true;
           
           // Check if "Other" option was selected
           if (radio.id === 'option4') {
             document.getElementById('otherReason').style.display = 'block';
           } else {
             document.getElementById('otherReason').style.display = 'none';
           }
         });
       });
       
       // Close functionality for both the X button and overlay
       function closeSurvey() {
         document.body.removeChild(surveyContainer);
         document.body.removeChild(overlay);
       }
       
       document.getElementById('closeButtonSurvey').addEventListener('click', closeSurvey);
       overlay.addEventListener('click', closeSurvey);
       
       // Submit functionality
       document.getElementById('submitButtonSurvey').addEventListener('click', function() {
         var selectedOption = document.querySelector('input[name="buttonSurvey"]:checked');
         
         if (selectedOption) {
           var response = selectedOption.id;
           var responseText = selectedOption.nextElementSibling.textContent;
           
           // If "Other" was selected, get the text
           if (response === 'option4') {
             responseText = document.getElementById('otherReasonText').value || "Other (no details provided)";
           }
           
           // Send the data
           dataLayer.push({
             'event': 'buttonSurveyResponse',
             'buttonType': buttonType,
             'buttonText': buttonText, 
             'responseOption': response,
             'responseText': responseText,
             'pagePath': window.location.pathname
           });
           
           // Thank user
           surveyContainer.innerHTML = '<div style="text-align:center; padding:30px;"><h3 style="margin-bottom:15px;">Thank You!</h3><p>Your feedback helps us improve our website.</p></div>';
           
           // Remove after delay
           setTimeout(function() {
             if (document.body.contains(surveyContainer)) {
               document.body.removeChild(surveyContainer);
               document.body.removeChild(overlay);
             }
           }, 2000);
         } else {
           alert('Please select an option');
         }
       });
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Button Survey Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `buttonSurveyTrigger`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your survey tag, select **Button Survey Trigger**.
9. Click **Save**.

---

## Step 5: Create Data Collection Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Button Survey Response Collector**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code (replace with your actual survey endpoint):

   ```html
   <script>
     (function() {
       // Get response data from dataLayer
       var dlEvent = {{dataLayer}};
       
       // Only proceed if this is a button survey response
       if (dlEvent.event !== 'buttonSurveyResponse') return;
       
       // Prepare data for sending
       var responseData = {
         type: 'buttonClickSurvey',
         buttonType: dlEvent.buttonType,
         buttonText: dlEvent.buttonText,
         responseOption: dlEvent.responseOption,
         responseText: dlEvent.responseText,
         pagePath: dlEvent.pagePath,
         url: window.location.href,
         timestamp: new Date().toISOString(),
         userId: {{User ID}} || 'anonymous'
       };
       
       // Send to your survey endpoint
       fetch('https://your-survey-endpoint.com/api/button-survey', {
         method: 'POST',
         headers: {
           'Content-Type': 'application/json',
         },
         body: JSON.stringify(responseData)
       })
       .then(response => response.json())
       .then(data => console.log('Button survey submitted:', data))
       .catch(error => console.error('Error submitting button survey:', error));
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Button Survey Response Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `buttonSurveyResponse`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your data collection tag, select **Button Survey Response Trigger**.
9. Click **Save**.

---

## Step 6: Variable Setup for DataLayer Values

1. Go to **Variables** in your GTM workspace.
2. Click the **New** button under "User-Defined Variables".
3. Name the variable: **dlv - buttonType**.
4. Click **Variable Configuration**.
5. Select **Data Layer Variable**.
6. Set **Data Layer Variable Name** to `buttonType`.
7. Click **Save**.

8. Repeat steps 2-7 to create a second variable:
   - Name: **dlv - buttonText**
   - Type: Data Layer Variable
   - Data Layer Variable Name: `buttonText`

---

## Step 7: Test and Publish

1. Click the **Preview** button in GTM.
2. Visit a page with your target buttons.
3. Click one of the buttons.
4. Verify that the survey appears.
5. Test submitting different responses.
6. Check that data is being collected properly.
7. If everything functions correctly, publish your changes.

---

## Optional Enhancements

- **Different Questions by Button Type:** Create different survey questions for different button types (purchase, download, signup) by further customizing the survey content based on buttonType.
- **A/B Testing Survey Timing:** Test different delay times to find the optimal moment to show the survey.
- **Survey Rate Limiting:** Implement more sophisticated logic to limit how many button click surveys a user sees per session.
- **Conditional Surveys:** Only show surveys for certain button clicks based on user behavior patterns or session attributes.
- **Progressive Surveys:** Ask different questions on each button click to build a more complete picture of user intent over time.

--- 