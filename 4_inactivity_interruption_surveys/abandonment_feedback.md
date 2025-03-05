# GTM Abandonment Feedback Survey Implementation Guide

This guide explains how to implement abandonment feedback surveys using Google Tag Manager (GTM) to gather insights from users who begin a process (like filling out a form or adding items to a cart) but then abandon it before completion.

---

## Step 1: Identify Abandonment Scenarios

Before implementation, determine which abandonment scenarios are most important to track:

1. Common abandonment scenarios:
   - Cart abandonment (items added but checkout not completed)
   - Form abandonment (form partially filled but not submitted)
   - Registration abandonment (signup process started but not completed)
   - Checkout abandonment (checkout process started but purchase not completed)
   - Content creation abandonment (started creating content but didn't publish)

2. Determine the criteria that define abandonment:
   - For forms: At least 1-2 fields filled out
   - For carts: At least one item added
   - For checkout: At least one step of multi-step checkout completed
   - For registrations: At least email or name entered

---

## Step 2: Create Abandonment Detection Logic

First, we'll create a tag to detect when a user has started a process that we want to track:

1. Go to the **Tags** section in your GTM workspace.
2. Click the **New** button.
3. Name the tag: **Process Start Detector**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Configuration
       var config = {
         processType: determineProcessType(),
         minEngagementThreshold: 1,  // Minimum fields filled or items added
         storageKey: 'pendingProcess'
       };
       
       // Skip if this page isn't a process we want to track
       if (config.processType === 'none') return;
       
       // Helper function to determine process type
       function determineProcessType() {
         var path = window.location.pathname;
         
         if (path.match(/\/cart\b/) || document.querySelector('.cart-page, .shopping-cart')) {
           return 'cart';
         } else if (path.match(/\/checkout\b/) || document.querySelector('.checkout-page, .checkout-form')) {
           return 'checkout';
         } else if (path.match(/\/register\b|\/signup\b|\/sign-up\b/) || document.querySelector('.register-form, .signup-form')) {
           return 'registration';
         } else if (document.querySelector('form') && document.querySelectorAll('form input, form select, form textarea').length >= 3) {
           return 'form';
         }
         
         return 'none';
       }
       
       // Set up listeners to detect engagement with the process
       function setUpEngagementListeners() {
         var engagementCount = 0;
         var processStarted = false;
         
         // Custom logic based on process type
         if (config.processType === 'form' || config.processType === 'registration') {
           // Monitor form field interactions
           document.querySelectorAll('input, select, textarea').forEach(function(field) {
             field.addEventListener('change', function() {
               if (field.value.trim().length > 0) {
                 engagementCount++;
                 checkEngagementThreshold();
               }
             });
           });
         } else if (config.processType === 'cart') {
           // Check if cart already has items
           var cartItemCount = document.querySelectorAll('.cart-item, .product-item, [data-product-id]').length;
           if (cartItemCount > 0) {
             engagementCount = cartItemCount;
             checkEngagementThreshold();
           }
           
           // Monitor add to cart buttons
           document.querySelectorAll('.add-to-cart, [data-add-to-cart]').forEach(function(button) {
             button.addEventListener('click', function() {
               engagementCount++;
               checkEngagementThreshold();
             });
           });
         } else if (config.processType === 'checkout') {
           // If we're on checkout, process is already started
           processStarted = true;
           storeProcessState();
         }
         
         // Check if we've met the threshold for considering the process started
         function checkEngagementThreshold() {
           if (!processStarted && engagementCount >= config.minEngagementThreshold) {
             processStarted = true;
             storeProcessState();
           }
         }
         
         // Store process state to detect abandonment
         function storeProcessState() {
           var processData = {
             type: config.processType,
             url: window.location.href,
             timestamp: new Date().toISOString(),
             pageTitle: document.title
           };
           
           // Store in sessionStorage
           try {
             sessionStorage.setItem(config.storageKey, JSON.stringify(processData));
           } catch (e) {
             console.error('Failed to store process state: ', e);
           }
           
           // Push to dataLayer
           dataLayer.push({
             'event': 'processStarted',
             'processType': config.processType
           });
         }
       }
       
       // Initialize monitoring
       setUpEngagementListeners();
     })();
   </script>
   ```

7. Under **Triggering**, select **All Pages** or create specific triggers for cart, checkout, and form pages.
8. Click **Save**.

---

## Step 3: Create Abandonment Detection Tag

Now, let's create a tag to detect when a user is about to leave without completing the process:

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Abandonment Detector**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Check if there's a pending process
       var storageKey = 'pendingProcess';
       var pendingProcess = null;
       
       try {
         var storedProcess = sessionStorage.getItem(storageKey);
         if (storedProcess) {
           pendingProcess = JSON.parse(storedProcess);
         }
       } catch (e) {
         console.error('Failed to retrieve process state: ', e);
         return;
       }
       
       // If no pending process, exit
       if (!pendingProcess) return;
       
       // Check if we're on a "completion" page that indicates the process was successful
       var currentPath = window.location.pathname;
       var isCompletionPage = false;
       
       // Define completion pages based on process type
       if (pendingProcess.type === 'cart' && currentPath.match(/\/checkout\b/)) {
         isCompletionPage = true;
       } else if (pendingProcess.type === 'checkout' && currentPath.match(/\/thank-you\b|\/confirmation\b|\/order-confirmed\b/)) {
         isCompletionPage = true;
       } else if ((pendingProcess.type === 'registration' || pendingProcess.type === 'form') && 
                  (currentPath.match(/\/thank-you\b|\/confirmation\b|\/success\b/) || 
                   document.querySelector('.success-message, .thank-you-message, .confirmation-message'))) {
         isCompletionPage = true;
       }
       
       // If we're on a completion page, clear the pending process
       if (isCompletionPage) {
         sessionStorage.removeItem(storageKey);
         return;
       }
       
       // Check if a cookie exists to prevent showing the survey multiple times
       var cookieName = 'abandonmentSurveyShown';
       if (document.cookie.indexOf(cookieName + '=') !== -1) return;
       
       // Track when user is about to leave
       var surveyShown = false;
       
       // Function to handle abandonment
       function handleAbandonment() {
         if (surveyShown) return;
         surveyShown = true;
         
         // Push event to dataLayer
         dataLayer.push({
           'event': 'processAbandoned',
           'processType': pendingProcess.type,
           'processStartUrl': pendingProcess.url,
           'processStartTime': pendingProcess.timestamp,
           'processDuration': Math.round((new Date() - new Date(pendingProcess.timestamp)) / 1000) // in seconds
         });
         
         // Set cookie to prevent showing too frequently
         document.cookie = cookieName + "=true; path=/; max-age=1800"; // 30 minutes
       }
       
       // Set up exit intent detection
       function setupExitIntent() {
         // Track mouse leaving the window (top of page)
         document.addEventListener('mouseleave', function(e) {
           if (e.clientY < 50) {
             handleAbandonment();
           }
         });
         
         // Track browser tab/window close or navigation away
         window.addEventListener('beforeunload', function() {
           handleAbandonment();
         });
         
         // Track back button usage
         window.addEventListener('popstate', function() {
           handleAbandonment();
         });
         
         // If the user is inactive for too long after starting a process
         setTimeout(function() {
           // Only trigger if they're still on the page and haven't shown the survey yet
           if (document.hasFocus() && !surveyShown) {
             handleAbandonment();
           }
         }, 60000); // 60 seconds of inactivity
       }
       
       // Initialize exit intent detection
       setupExitIntent();
     })();
   </script>
   ```

7. Under **Triggering**, select **All Pages**.
8. Click **Save**.

---

## Step 4: Create Abandonment Survey Display Tag

Now, let's create a tag to display the abandonment feedback survey:

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Abandonment Survey Display**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Get abandonment information from dataLayer
       var processType = {{dlv - processType}} || 'process';
       var startUrl = {{dlv - processStartUrl}} || window.location.href;
       var processDuration = {{dlv - processDuration}} || 0;
       
       // Helper function to create and display the survey
       function createSurvey() {
         // Prevent page navigation
         var canNavigateAway = false;
         
         // Create survey container
         var surveyContainer = document.createElement('div');
         surveyContainer.className = 'abandonment-feedback-survey';
         surveyContainer.style.cssText = 'position:fixed; top:50%; left:50%; transform:translate(-50%, -50%); width:450px; background:#fff; border-radius:10px; box-shadow:0 0 25px rgba(0,0,0,0.4); padding:25px; z-index:999999; font-family:Arial,sans-serif;';
         
         // Create background overlay
         var overlay = document.createElement('div');
         overlay.style.cssText = 'position:fixed; top:0; left:0; width:100%; height:100%; background:rgba(0,0,0,0.6); z-index:999998;';
         
         // Customize survey based on process type
         var surveyTitle = "Wait! Before You Go...";
         var surveyQuestion = "What stopped you from completing this process?";
         var surveyOptions = [];
         
         if (processType === 'cart') {
           surveyTitle = "Wait! You Left Items in Your Cart";
           surveyQuestion = "What stopped you from completing your purchase?";
           surveyOptions = [
             { value: 'just-browsing', text: "I'm just browsing" },
             { value: 'price-shipping', text: "Shipping costs or total price" },
             { value: 'payment-options', text: "Payment options" },
             { value: 'need-more-info', text: "Need more product information" },
             { value: 'technical-issue', text: "I experienced a technical issue" },
             { value: 'other', text: "Other reason" }
           ];
         } else if (processType === 'checkout') {
           surveyTitle = "Wait! Your Order is Not Complete";
           surveyQuestion = "What stopped you from completing your purchase?";
           surveyOptions = [
             { value: 'second-thoughts', text: "Changed my mind" },
             { value: 'shipping-cost', text: "Shipping costs were too high" },
             { value: 'payment-issue', text: "Payment issues" },
             { value: 'technical-issue', text: "Technical problems on the site" },
             { value: 'comparison', text: "Comparing with other sites" },
             { value: 'other', text: "Other reason" }
           ];
         } else if (processType === 'registration') {
           surveyTitle = "Wait! Registration Not Complete";
           surveyQuestion = "What stopped you from completing your registration?";
           surveyOptions = [
             { value: 'too-much-info', text: "Asked for too much information" },
             { value: 'privacy-concerns', text: "Privacy concerns" },
             { value: 'complex-process', text: "Process too complicated" },
             { value: 'technical-issue', text: "Technical issues" },
             { value: 'other', text: "Other reason" }
           ];
         } else if (processType === 'form') {
           surveyTitle = "Wait! Form Not Submitted";
           surveyQuestion = "What stopped you from submitting the form?";
           surveyOptions = [
             { value: 'too-long', text: "Form is too long" },
             { value: 'confusing', text: "Questions are confusing" },
             { value: 'privacy-concerns', text: "Privacy concerns" },
             { value: 'technical-issue', text: "Technical issues" },
             { value: 'other', text: "Other reason" }
           ];
         }
         
         // Build options HTML
         var optionsHTML = '';
         surveyOptions.forEach(function(option) {
           optionsHTML += `
             <div class="abandonment-option" data-value="${option.value}" style="padding:12px; background:#f5f5f5; border:none; border-radius:5px; cursor:pointer; font-size:15px; margin-bottom:8px; text-align:left;">
               ${option.text}
             </div>
           `;
         });
         
         // Survey content
         surveyContainer.innerHTML = `
           <div style="text-align:center; margin-bottom:20px;">
             <h2 style="margin:0 0 10px; color:#333; font-size:22px;">${surveyTitle}</h2>
           </div>
           <p style="margin:0 0 20px; color:#444; font-size:16px; text-align:center;">${surveyQuestion}</p>
           <div style="display:flex; flex-direction:column; margin-bottom:20px;">
             ${optionsHTML}
           </div>
           <div id="abandonmentFeedback" style="display:none; margin-bottom:20px;">
             <textarea id="abandonmentFeedbackText" placeholder="Please tell us more..." style="width:100%; padding:12px; border:1px solid #ddd; border-radius:5px; resize:vertical; min-height:80px; box-sizing:border-box; font-family:inherit;"></textarea>
             <button id="submitAbandonmentSurvey" style="width:100%; margin-top:15px; padding:12px; background:#4a90e2; color:#fff; border:none; border-radius:5px; cursor:pointer; font-weight:bold; font-size:16px;">Submit Feedback</button>
           </div>
           <div style="display:flex; justify-content:space-between; align-items:center;">
             <button id="continueProcess" style="padding:10px 15px; background:#4a90e2; color:#fff; border:none; border-radius:5px; cursor:pointer; font-weight:bold;">Continue Where I Left Off</button>
             <button id="closeAbandonmentSurvey" style="background:none; border:none; color:#999; font-size:14px; cursor:pointer; text-decoration:underline;">Skip</button>
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
           
           // Allow navigation away
           canNavigateAway = true;
           
           // Clear the stored process state
           try {
             sessionStorage.removeItem('pendingProcess');
           } catch (e) {
             console.error('Failed to clear process state: ', e);
           }
         }
         
         // Handle "continue where I left off" button
         document.getElementById('continueProcess').addEventListener('click', function() {
           // Navigate back to the process
           canNavigateAway = true;
           window.location.href = startUrl;
         });
         
         // Close button functionality
         document.getElementById('closeAbandonmentSurvey').addEventListener('click', closeSurvey);
         
         // Option button functionality
         var optionButtons = document.querySelectorAll('.abandonment-option');
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
             
             // Show feedback text area for "other" option
             if (selectedOption === 'other' || selectedOption === 'technical-issue') {
               document.getElementById('abandonmentFeedback').style.display = 'block';
             } else {
               // For other options, submit immediately
               submitFeedback();
             }
           });
         });
         
         // Submit button functionality
         var submitButton = document.getElementById('submitAbandonmentSurvey');
         if (submitButton) {
           submitButton.addEventListener('click', function() {
             submitFeedback();
           });
         }
         
         // Function to submit feedback
         function submitFeedback() {
           if (!selectedOption) return;
           
           var feedbackText = '';
           var feedbackTextArea = document.getElementById('abandonmentFeedbackText');
           if (feedbackTextArea && feedbackTextArea.value) {
             feedbackText = feedbackTextArea.value;
           }
           
           // Send the data
           dataLayer.push({
             'event': 'abandonmentSurveyResponse',
             'processType': processType,
             'processDuration': processDuration,
             'surveyResponse': selectedOption,
             'surveyFeedback': feedbackText
           });
           
           // Show thank you message
           surveyContainer.innerHTML = '<div style="text-align:center; padding:20px;"><h3 style="margin-bottom:15px; color:#333;">Thank You!</h3><p style="color:#666;">Your feedback helps us improve our service.</p></div>';
           
           // Update specific messages or offers based on the feedback
           if (processType === 'cart' && (selectedOption === 'price-shipping' || selectedOption === 'payment-options')) {
             setTimeout(function() {
               surveyContainer.innerHTML = `
                 <div style="text-align:center; padding:20px;">
                   <h3 style="margin-bottom:15px; color:#333;">Special Offer Just For You!</h3>
                   <p style="color:#666; margin-bottom:15px;">Would you like to complete your purchase with free shipping?</p>
                   <button onclick="window.location.href='${startUrl}'" style="padding:12px 20px; background:#4a90e2; color:#fff; border:none; border-radius:5px; cursor:pointer; font-weight:bold; margin-bottom:10px; width:100%;">Yes, Continue With Free Shipping</button>
                   <button onclick="closeSurvey()" style="background:none; border:none; color:#999; font-size:14px; cursor:pointer; text-decoration:underline;">No thanks</button>
                 </div>
               `;
             }, 1500);
           } else {
             // Remove after delay
             setTimeout(function() {
               closeSurvey();
             }, 3000);
           }
         }
         
         // Prevent navigation away before survey is completed
         window.addEventListener('beforeunload', function(e) {
           if (!canNavigateAway) {
             var confirmationMessage = 'Wait! Please complete the quick survey before leaving.';
             (e || window.event).returnValue = confirmationMessage;
             return confirmationMessage;
           }
         });
       }
       
       // Display the survey after a small delay
       setTimeout(createSurvey, 100);
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Process Abandonment Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `processAbandoned`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your survey tag, select **Process Abandonment Trigger**.
9. Click **Save**.

---

## Step 5: Create Data Collection Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Abandonment Survey Response Collector**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code (replace with your actual survey endpoint):

   ```html
   <script>
     (function() {
       // Get response data from dataLayer
       var dlEvent = {{dataLayer}};
       
       // Only proceed if this is an abandonment survey response
       if (dlEvent.event !== 'abandonmentSurveyResponse') return;
       
       // Prepare data for sending
       var responseData = {
         type: 'abandonmentSurvey',
         processType: dlEvent.processType,
         processDuration: dlEvent.processDuration,
         response: dlEvent.surveyResponse,
         feedback: dlEvent.surveyFeedback,
         url: window.location.href,
         timestamp: new Date().toISOString(),
         userId: {{User ID}} || 'anonymous'
       };
       
       // Send to your survey endpoint
       fetch('https://your-survey-endpoint.com/api/abandonment-survey', {
         method: 'POST',
         headers: {
           'Content-Type': 'application/json',
         },
         body: JSON.stringify(responseData)
       })
       .then(response => response.json())
       .then(data => console.log('Abandonment survey submitted:', data))
       .catch(error => console.error('Error submitting abandonment survey:', error));
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Abandonment Survey Response Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `abandonmentSurveyResponse`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your data collection tag, select **Abandonment Survey Response Trigger**.
9. Click **Save**.

---

## Step 6: Variable Setup for DataLayer Values

1. Go to **Variables** in your GTM workspace.
2. Click the **New** button under "User-Defined Variables".
3. Create the following variables:

   a. **dlv - processType**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `processType`

   b. **dlv - processStartUrl**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `processStartUrl`

   c. **dlv - processDuration**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `processDuration`

4. Click **Save** for each variable.

---

## Step 7: Test and Publish

1. Click the **Preview** button in GTM.
2. Visit a page with a process you're tracking (cart, checkout, form, etc.).
3. Begin the process but don't complete it (add an item to cart, start filling out a form, etc.).
4. Navigate away from the page or simulate exit intent by moving your cursor to the top of the browser.
5. Verify that the abandonment survey appears.
6. Test each response option.
7. Verify that the "Continue Where I Left Off" button works.
8. Verify data is being collected properly.
9. If everything functions correctly, publish your changes.

---

## Optional Enhancements

- **Personalized Incentives:** Offer personalized incentives based on abandonment reason (discount code for price concerns, support chat for technical issues).
- **Progressive Retargeting:** Adjust follow-up messaging based on survey responses.
- **Segment-Based Analysis:** Analyze abandonment patterns by user segments to identify patterns.
- **Multi-Device Tracking:** Use User ID or other techniques to track abandonment across devices.
- **Adaptive Survey Questions:** Change the questions based on where in the process the user abandoned (early vs. late abandonment).

---

## Advanced: Cart Recovery Email Integration

If you collect an email address before abandonment, you can integrate this with an email recovery system:

```javascript
// Enhanced abandonment handler with email recovery
function handleAbandonmentWithEmail() {
  // Regular abandonment handling...
  
  // Check if we have user's email
  var userEmail = getUserEmail();
  
  if (userEmail) {
    // Add to email recovery flow
    fetch('https://your-api.com/cart-recovery', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        email: userEmail,
        cartItems: getCartItems(),
        abandonedUrl: window.location.href,
        timestamp: new Date().toISOString()
      })
    })
    .then(response => response.json())
    .then(data => console.log('Email recovery flow initiated'))
    .catch(error => console.error('Error setting up email recovery:', error));
  }
}

// Helper to find user email from form fields, logged in state, etc.
function getUserEmail() {
  // Check for email in form fields
  var emailFields = document.querySelectorAll('input[type="email"], input[name*="email"]');
  for (var i = 0; i < emailFields.length; i++) {
    if (emailFields[i].value && emailFields[i].value.match(/.+@.+\..+/)) {
      return emailFields[i].value;
    }
  }
  
  // Check for logged in user data
  if (window.userData && window.userData.email) {
    return window.userData.email;
  }
  
  // Check localStorage/cookies
  try {
    var storedUser = localStorage.getItem('user');
    if (storedUser) {
      var userData = JSON.parse(storedUser);
      if (userData.email) return userData.email;
    }
  } catch (e) {
    console.error('Error retrieving stored user data', e);
  }
  
  return null;
}

// Get items in cart
function getCartItems() {
  // This would be customized based on your site's structure
  var items = [];
  
  document.querySelectorAll('.cart-item, .product-item').forEach(function(item) {
    items.push({
      id: item.getAttribute('data-product-id') || 'unknown',
      name: item.querySelector('.product-name, .item-name')?.textContent || 'Unknown product',
      price: item.querySelector('.product-price, .item-price')?.textContent || 'Unknown price',
      quantity: item.querySelector('.quantity-input, .item-quantity')?.value || 1
    });
  });
  
  return items;
}
```

This advanced implementation helps recover lost sales by sending targeted emails to users who abandon their carts or other processes.

---
``` 