# GTM Post-Purchase/Sign-Up Survey Implementation Guide

This guide explains how to implement post-conversion surveys using Google Tag Manager (GTM) to gather feedback immediately after a user completes a purchase or registration process.

---

## Step 1: Identify Conversion Pages

Before implementation, identify the pages that indicate a successful conversion:

1. Common conversion pages:
   - Order confirmation pages
   - Registration confirmation pages
   - Subscription confirmation pages
   - Account creation success pages
   - Download completion pages

2. Typical URL patterns:
   - `/thank-you`
   - `/order-confirmation`
   - `/registration-complete`
   - `/success`
   - `/download-complete`

---

## Step 2: Create Conversion Page View Trigger

1. Navigate to **Triggers** in your GTM workspace.
2. Click the **New** button.
3. Name the trigger: **Conversion Page View Trigger**.
4. Click **Trigger Configuration**.
5. Select **Page View**.
6. Choose **Some Page Views**.
7. Set up conditions for your conversion pages:
   - **Page Path** contains `/thank-you` OR
   - **Page Path** contains `/order-confirmed` OR
   - **Page Path** contains `/registration-complete` OR
   - **Page Path** contains `/success`
   
   (Customize these paths based on your website's structure)
8. Click **Save**.

---

## Step 3: Create Survey Delay Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Post-Conversion Survey Delay**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Only set timeout once
       if (window.conversionSurveyDelayStarted) return;
       window.conversionSurveyDelayStarted = true;
       
       // Get conversion type from URL
       var conversionType = determineConversionType();
       
       // Get transaction/conversion data if available
       var conversionData = getConversionData();
       
       // Store conversion data for survey
       window.conversionSurveyData = {
         type: conversionType,
         data: conversionData,
         page: window.location.pathname,
         referrer: document.referrer
       };
       
       // Determine appropriate delay time
       var delayTime = 3000; // 3 seconds (enough to let the user see success message)
       
       // Check if survey already shown in past 7 days
       var cookieName = 'postConversionSurveyShown';
       if (document.cookie.indexOf(cookieName + '=') === -1) {
         // Wait before triggering the survey
         setTimeout(function() {
           // Check if user is still on page
           if (document.hidden) return;
           
           // Fire the custom event after delay
           dataLayer.push({
             'event': 'showPostConversionSurvey',
             'conversionType': conversionType,
             'conversionValue': conversionData.value || 0,
             'conversionID': conversionData.id || ''
           });
           
           // Set cookie to prevent showing too frequently
           document.cookie = cookieName + "=true; path=/; max-age=604800"; // 7 days
         }, delayTime);
       }
       
       // Helper function to determine conversion type
       function determineConversionType() {
         var path = window.location.pathname.toLowerCase();
         
         if (path.includes('order') || path.includes('purchase') || path.includes('checkout')) {
           return 'purchase';
         } else if (path.includes('register') || path.includes('signup') || path.includes('sign-up')) {
           return 'registration';
         } else if (path.includes('subscribe') || path.includes('subscription')) {
           return 'subscription';
         } else if (path.includes('download')) {
           return 'download';
         }
         
         // Default
         return 'conversion';
       }
       
       // Helper function to get conversion data
       function getConversionData() {
         var data = {
           value: 0,
           id: '',
           items: []
         };
         
         // Try to get transaction data from dataLayer
         try {
           // For purchase events (from Enhanced Ecommerce)
           if (window.dataLayer) {
             for (var i = 0; i < window.dataLayer.length; i++) {
               var dlItem = window.dataLayer[i];
               
               // Check for purchase event
               if (dlItem.event === 'purchase' && dlItem.ecommerce && dlItem.ecommerce.purchase) {
                 data.value = dlItem.ecommerce.purchase.actionField.revenue || 0;
                 data.id = dlItem.ecommerce.purchase.actionField.id || '';
                 data.items = dlItem.ecommerce.purchase.products || [];
                 return data;
               }
               
               // Also check for GA4 style purchase
               if (dlItem.event === 'purchase' && dlItem.ecommerce && dlItem.ecommerce.transaction_id) {
                 data.value = dlItem.ecommerce.value || 0;
                 data.id = dlItem.ecommerce.transaction_id || '';
                 data.items = dlItem.ecommerce.items || [];
                 return data;
               }
             }
           }
           
           // Try to find order values in the DOM (fallback)
           var orderIdElement = document.querySelector('.order-id, .confirmation-number, [data-order-id]');
           if (orderIdElement) {
             data.id = orderIdElement.textContent.trim() || orderIdElement.getAttribute('data-order-id');
           }
           
           var orderValueElement = document.querySelector('.order-total, .total-amount, [data-order-value]');
           if (orderValueElement) {
             var valueText = orderValueElement.textContent.trim();
             // Extract number from string (e.g., "$123.45" → 123.45)
             var valueMatch = valueText.match(/[\d,.]+/);
             if (valueMatch) {
               data.value = parseFloat(valueMatch[0].replace(/,/g, ''));
             }
           }
         } catch (e) {
           console.error('Error getting conversion data:', e);
         }
         
         return data;
       }
     })();
   </script>
   ```

7. Under **Triggering**, select your **Conversion Page View Trigger**.
8. Click **Save**.

---

## Step 4: Create Survey Display Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Post-Conversion Survey Display**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Get conversion information from dataLayer
       var conversionType = {{dlv - conversionType}} || 'purchase';
       var conversionValue = {{dlv - conversionValue}} || 0;
       var conversionID = {{dlv - conversionID}} || '';
       
       // Create survey container
       var surveyContainer = document.createElement('div');
       surveyContainer.className = 'post-conversion-survey';
       surveyContainer.style.cssText = 'position:fixed; top:50%; left:50%; transform:translate(-50%, -50%); width:450px; background:#fff; border-radius:10px; box-shadow:0 0 25px rgba(0,0,0,0.3); padding:25px; z-index:999999; font-family:Arial,sans-serif;';
       
       // Create background overlay
       var overlay = document.createElement('div');
       overlay.style.cssText = 'position:fixed; top:0; left:0; width:100%; height:100%; background:rgba(0,0,0,0.5); z-index:999998;';
       
       // Customize survey based on conversion type
       var surveyTitle = "Quick Feedback";
       var surveyQuestion = "How was your experience today?";
       var surveyQuestionId = "general_experience";
       var additionalQuestions = [];
       
       if (conversionType === 'purchase') {
         surveyTitle = "Thank You for Your Purchase!";
         surveyQuestion = "How would you rate your purchase experience?";
         surveyQuestionId = "purchase_experience";
         additionalQuestions = [
           { id: "ease_of_checkout", text: "How easy was the checkout process?" },
           { id: "product_selection", text: "How satisfied are you with our product selection?" }
         ];
       } else if (conversionType === 'registration') {
         surveyTitle = "Welcome to Our Community!";
         surveyQuestion = "How easy was the registration process?";
         surveyQuestionId = "registration_experience";
         additionalQuestions = [
           { id: "clarity_of_form", text: "How clear were the registration instructions?" }
         ];
       } else if (conversionType === 'subscription') {
         surveyTitle = "Thanks for Subscribing!";
         surveyQuestion = "How would you rate the subscription process?";
         surveyQuestionId = "subscription_experience";
       } else if (conversionType === 'download') {
         surveyTitle = "Thank You for Your Download!";
         surveyQuestion = "How easy was it to find what you needed?";
         surveyQuestionId = "download_experience";
       }
       
       // Build the survey content HTML
       var surveyHTML = `
         <div style="text-align:center; margin-bottom:25px;">
           <h2 style="margin:0 0 10px; color:#333; font-size:22px;">${surveyTitle}</h2>
           <p style="margin:0; color:#666; font-size:14px;">We'd love to hear your feedback!</p>
         </div>
         <div style="margin-bottom:25px;">
           <p style="margin:0 0 15px; color:#444; font-size:16px;">${surveyQuestion}</p>
           <div style="display:flex; justify-content:space-between; gap:5px; margin-bottom:20px;">
             <button class="rating-btn" data-id="${surveyQuestionId}" data-rating="1" style="flex:1; padding:10px 5px; background:#f9f9f9; border:1px solid #ddd; border-radius:5px; cursor:pointer;">1<br><span style="font-size:11px;">Poor</span></button>
             <button class="rating-btn" data-id="${surveyQuestionId}" data-rating="2" style="flex:1; padding:10px 5px; background:#f9f9f9; border:1px solid #ddd; border-radius:5px; cursor:pointer;">2</button>
             <button class="rating-btn" data-id="${surveyQuestionId}" data-rating="3" style="flex:1; padding:10px 5px; background:#f9f9f9; border:1px solid #ddd; border-radius:5px; cursor:pointer;">3</button>
             <button class="rating-btn" data-id="${surveyQuestionId}" data-rating="4" style="flex:1; padding:10px 5px; background:#f9f9f9; border:1px solid #ddd; border-radius:5px; cursor:pointer;">4</button>
             <button class="rating-btn" data-id="${surveyQuestionId}" data-rating="5" style="flex:1; padding:10px 5px; background:#f9f9f9; border:1px solid #ddd; border-radius:5px; cursor:pointer;">5<br><span style="font-size:11px;">Excellent</span></button>
           </div>
         </div>
       `;
       
       // Add additional questions if applicable
       if (additionalQuestions.length > 0) {
         additionalQuestions.forEach(function(question) {
           surveyHTML += `
             <div class="additional-question" style="display:none; margin-bottom:20px;">
               <p style="margin:0 0 10px; color:#444; font-size:15px;">${question.text}</p>
               <div style="display:flex; justify-content:space-between; gap:5px;">
                 <button class="rating-btn" data-id="${question.id}" data-rating="1" style="flex:1; padding:8px 5px; background:#f9f9f9; border:1px solid #ddd; border-radius:5px; cursor:pointer;">1</button>
                 <button class="rating-btn" data-id="${question.id}" data-rating="2" style="flex:1; padding:8px 5px; background:#f9f9f9; border:1px solid #ddd; border-radius:5px; cursor:pointer;">2</button>
                 <button class="rating-btn" data-id="${question.id}" data-rating="3" style="flex:1; padding:8px 5px; background:#f9f9f9; border:1px solid #ddd; border-radius:5px; cursor:pointer;">3</button>
                 <button class="rating-btn" data-id="${question.id}" data-rating="4" style="flex:1; padding:8px 5px; background:#f9f9f9; border:1px solid #ddd; border-radius:5px; cursor:pointer;">4</button>
                 <button class="rating-btn" data-id="${question.id}" data-rating="5" style="flex:1; padding:8px 5px; background:#f9f9f9; border:1px solid #ddd; border-radius:5px; cursor:pointer;">5</button>
               </div>
             </div>
           `;
         });
       }
       
       // Add feedback text area and submit button
       surveyHTML += `
         <div id="feedbackSection" style="display:none; margin-bottom:20px;">
           <p style="margin:0 0 10px; color:#444; font-size:15px;">Would you like to share any additional feedback?</p>
           <textarea id="additionalFeedback" style="width:100%; padding:12px; border:1px solid #ddd; border-radius:5px; resize:vertical; min-height:80px; box-sizing:border-box;"></textarea>
         </div>
         <div style="display:flex; justify-content:space-between; align-items:center;">
           <button id="submitConversionSurvey" style="padding:12px 20px; background:#4a90e2; color:#fff; border:none; border-radius:5px; cursor:pointer; font-weight:bold; display:none;">Submit Feedback</button>
           <button id="closeConversionSurvey" style="background:none; border:none; color:#999; font-size:14px; cursor:pointer; text-decoration:underline;">No thanks</button>
         </div>
       `;
       
       // Set the survey container content
       surveyContainer.innerHTML = surveyHTML;
       
       // Add to page
       document.body.appendChild(overlay);
       document.body.appendChild(surveyContainer);
       
       // Prevent page scrolling when survey is open
       document.body.style.overflow = 'hidden';
       
       // Store the responses
       var surveyResponses = {
         conversionType: conversionType,
         conversionID: conversionID,
         conversionValue: conversionValue,
         ratings: {},
         feedback: ''
       };
       
       // Close survey function
       function closeSurvey() {
         document.body.removeChild(surveyContainer);
         document.body.removeChild(overlay);
         document.body.style.overflow = '';
       }
       
       // Close button functionality
       document.getElementById('closeConversionSurvey').addEventListener('click', closeSurvey);
       
       // Rating button functionality for primary question
       var primaryRatingButtons = surveyContainer.querySelectorAll(`.rating-btn[data-id="${surveyQuestionId}"]`);
       primaryRatingButtons.forEach(function(button) {
         button.addEventListener('click', function() {
           // Get the rating value
           var rating = this.getAttribute('data-rating');
           var questionId = this.getAttribute('data-id');
           
           // Store the response
           surveyResponses.ratings[questionId] = rating;
           
           // Reset all buttons
           primaryRatingButtons.forEach(function(btn) {
             btn.style.background = '#f9f9f9';
             btn.style.borderColor = '#ddd';
           });
           
           // Highlight selected button
           this.style.background = '#e0e7ff';
           this.style.borderColor = '#4a90e2';
           
           // Show additional questions
           var additionalQuestions = surveyContainer.querySelectorAll('.additional-question');
           if (additionalQuestions.length > 0) {
             additionalQuestions[0].style.display = 'block';
           } else {
             // If no additional questions, show feedback section
             document.getElementById('feedbackSection').style.display = 'block';
             document.getElementById('submitConversionSurvey').style.display = 'block';
           }
         });
       });
       
       // Rating button functionality for additional questions
       var additionalQuestions = surveyContainer.querySelectorAll('.additional-question');
       additionalQuestions.forEach(function(questionDiv, index) {
         var buttons = questionDiv.querySelectorAll('.rating-btn');
         buttons.forEach(function(button) {
           button.addEventListener('click', function() {
             // Get the rating value
             var rating = this.getAttribute('data-rating');
             var questionId = this.getAttribute('data-id');
             
             // Store the response
             surveyResponses.ratings[questionId] = rating;
             
             // Reset all buttons in this question
             buttons.forEach(function(btn) {
               btn.style.background = '#f9f9f9';
               btn.style.borderColor = '#ddd';
             });
             
             // Highlight selected button
             this.style.background = '#e0e7ff';
             this.style.borderColor = '#4a90e2';
             
             // Show next question or feedback section
             if (index < additionalQuestions.length - 1) {
               additionalQuestions[index + 1].style.display = 'block';
             } else {
               // Last question answered, show feedback section
               document.getElementById('feedbackSection').style.display = 'block';
               document.getElementById('submitConversionSurvey').style.display = 'block';
             }
           });
         });
       });
       
       // Submit button functionality
       document.getElementById('submitConversionSurvey').addEventListener('click', function() {
         // Get additional feedback
         surveyResponses.feedback = document.getElementById('additionalFeedback').value || '';
         
         // Send the data
         dataLayer.push({
           'event': 'postConversionSurveyResponse',
           'conversionType': surveyResponses.conversionType,
           'conversionID': surveyResponses.conversionID,
           'conversionValue': surveyResponses.conversionValue,
           'surveyRatings': JSON.stringify(surveyResponses.ratings),
           'surveyFeedback': surveyResponses.feedback
         });
         
         // Show thank you message
         showThankYou();
       });
       
       // Function to show thank you message
       function showThankYou() {
         surveyContainer.innerHTML = `
           <div style="text-align:center; padding:30px;">
             <h3 style="margin-bottom:15px; color:#333;">Thank You for Your Feedback!</h3>
             <p style="color:#666; margin-bottom:20px;">We appreciate you taking the time to share your thoughts.</p>
             <button id="closeThankYou" style="padding:10px 20px; background:#4a90e2; color:#fff; border:none; border-radius:5px; cursor:pointer;">Continue</button>
           </div>
         `;
         
         // Add close functionality to continue button
         document.getElementById('closeThankYou').addEventListener('click', closeSurvey);
       }
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Show Post-Conversion Survey Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `showPostConversionSurvey`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your survey tag, select **Show Post-Conversion Survey Trigger**.
9. Click **Save**.

---

## Step 5: Create Data Collection Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Post-Conversion Survey Response Collector**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code (replace with your actual survey endpoint):

   ```html
   <script>
     (function() {
       // Get response data from dataLayer
       var dlEvent = {{dataLayer}};
       
       // Only proceed if this is a post-conversion survey response
       if (dlEvent.event !== 'postConversionSurveyResponse') return;
       
       // Parse the ratings JSON
       var ratings = {};
       try {
         ratings = JSON.parse(dlEvent.surveyRatings || '{}');
       } catch (e) {
         console.error('Error parsing survey ratings:', e);
       }
       
       // Prepare data for sending
       var responseData = {
         type: 'postConversionSurvey',
         conversionType: dlEvent.conversionType,
         conversionID: dlEvent.conversionID,
         conversionValue: dlEvent.conversionValue,
         ratings: ratings,
         feedback: dlEvent.surveyFeedback,
         url: window.location.href,
         timestamp: new Date().toISOString(),
         userId: {{User ID}} || 'anonymous'
       };
       
       // Send to your survey endpoint
       fetch('https://your-survey-endpoint.com/api/post-conversion-survey', {
         method: 'POST',
         headers: {
           'Content-Type': 'application/json',
         },
         body: JSON.stringify(responseData)
       })
       .then(response => response.json())
       .then(data => console.log('Post-conversion survey submitted:', data))
       .catch(error => console.error('Error submitting post-conversion survey:', error));
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Post-Conversion Survey Response Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `postConversionSurveyResponse`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your data collection tag, select **Post-Conversion Survey Response Trigger**.
9. Click **Save**.

---

## Step 6: Variable Setup for DataLayer Values

1. Go to **Variables** in your GTM workspace.
2. Click the **New** button under "User-Defined Variables".
3. Create the following variables:

   a. **dlv - conversionType**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `conversionType`

   b. **dlv - conversionValue**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `conversionValue`

   c. **dlv - conversionID**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `conversionID`

4. Click **Save** for each variable.

---

## Step 7: Test and Publish

1. Click the **Preview** button in GTM.
2. Complete a conversion process on your site (make a purchase, register, etc.).
3. Verify that after the configured delay, the survey appears.
4. Test the survey functionality:
   - Select different ratings
   - Check that additional questions appear
   - Enter feedback text
   - Submit the survey
5. Verify that the data is being collected properly.
6. If everything functions correctly, publish your changes.

---

## Optional Enhancements

- **Sampling Rate:** Only show the survey to a percentage of users to prevent survey fatigue.
   ```javascript
   // At the beginning of the delay tag
   var samplingRate = 0.25; // Show to 25% of users
   if (Math.random() > samplingRate) return;
   ```

- **Recent Customer Identification:** Customize surveys for returning customers.
   ```javascript
   // Add to the conversion data function
   var isReturningCustomer = {{Customer Status}} === 'returning';
   if (isReturningCustomer) {
     // Different questions for returning customers
   }
   ```

- **Multi-page Post-Conversion Journey:** Consider the full post-conversion journey for timing.
   ```javascript
   // Store in localStorage instead of cookies for multi-page tracking
   localStorage.setItem('conversionCompleted', JSON.stringify({
     type: conversionType,
     time: new Date().toISOString(),
     id: conversionID
   }));
   ```

- **Integrate with CRM:** Send survey responses along with conversion data to your CRM.
   ```javascript
   // Enhanced data collection with CRM integration
   function sendToCRM(data) {
     // Implementation depends on your CRM API
   }
   ```

- **NPS Survey Option:** Add a Net Promoter Score question for important conversions.
   ```javascript
   // Add to the survey HTML
   if (conversionType === 'purchase' && conversionValue > 100) {
     surveyHTML += `
       <div class="nps-question" style="margin-top:20px;">
         <p>How likely are you to recommend us to a friend or colleague? (0-10)</p>
         <div class="nps-scale">
           <!-- NPS scale buttons 0-10 -->
         </div>
       </div>
     `;
   }
   ```

---

## Advanced: Conditional Questions Based on Purchase Value or Type

For a more targeted approach, you can customize survey questions based on the specific details of the purchase:

```javascript
// In the survey display tag
function getCustomQuestionsForPurchase(conversionValue, conversionID) {
  // Get purchase details from dataLayer or DOM
  var productCategories = extractProductCategories();
  var questions = [];
  
  // High-value purchases
  if (conversionValue > 500) {
    questions.push({
      id: "purchase_decision_factors",
      text: "What was the most important factor in your decision to purchase?"
    });
  }
  
  // Product category specific questions
  if (productCategories.includes('electronics')) {
    questions.push({
      id: "tech_specs_clarity",
      text: "How clear was the technical information provided?"
    });
  }
  
  if (productCategories.includes('clothing')) {
    questions.push({
      id: "size_fit_accuracy",
      text: "How accurate was the size/fit information?"
    });
  }
  
  return questions;
}

function extractProductCategories() {
  // Implementation to extract categories from dataLayer or DOM
  var categories = [];
  // ...
  return categories;
}
```

By customizing questions based on purchase type, you'll gather more relevant feedback for specific product lines or service offerings.

--- 