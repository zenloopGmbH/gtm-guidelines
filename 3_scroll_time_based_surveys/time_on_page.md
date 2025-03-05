# GTM Time on Page Survey Implementation Guide

This guide explains how to implement time-based surveys using Google Tag Manager (GTM) to gather feedback from users who have spent a significant amount of time on your website, indicating engagement with your content.

---

## Step 1: Determine Appropriate Time Thresholds

Before implementation, decide which time thresholds would be appropriate for triggering a survey:

1. Common time thresholds to consider:
   - 30 seconds (basic engagement)
   - 60 seconds (meaningful engagement)
   - 2 minutes (deep engagement)
   - 5+ minutes (thorough content consumption)

2. Consider different thresholds for different content types:
   - For landing pages: 30-45 seconds may be sufficient
   - For blog articles: 90+ seconds indicates meaningful reading
   - For documentation: 2+ minutes shows serious interest
   - For product pages: 60+ seconds suggests consideration

---

## Step 2: Create Timer Trigger Tag

1. Go to the **Tags** section in your GTM workspace.
2. Click the **New** button.
3. Name the tag: **Time on Page Timer**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Check if timer already started
       if (window.timeOnPageTimerStarted) return;
       window.timeOnPageTimerStarted = true;
       
       // Check if survey already shown
       var cookieName = 'timeOnPageSurveyShown';
       if (document.cookie.indexOf(cookieName + '=') !== -1) return;
       
       // Determine appropriate time threshold based on page type
       var pageType = {{Page Type}} || 'general';
       var timeThreshold = 60000; // Default: 60 seconds
       
       if (pageType === 'blog' || pageType === 'article') {
         timeThreshold = 90000; // 90 seconds for blog/article
       } else if (pageType === 'documentation') {
         timeThreshold = 120000; // 2 minutes for documentation
       } else if (pageType === 'product') {
         timeThreshold = 60000; // 60 seconds for product pages
       } else if (pageType === 'landing') {
         timeThreshold = 30000; // 30 seconds for landing pages
       }
       
       // Store page data for later use
       window.timeOnPageData = {
         pageType: pageType,
         title: document.title,
         url: window.location.href,
         path: window.location.pathname,
         timeThreshold: timeThreshold
       };
       
       // Set timer to trigger the survey after the threshold
       window.timeOnPageTimer = setTimeout(function() {
         // Push event to dataLayer
         dataLayer.push({
           'event': 'timeOnPageSurveyTrigger',
           'pageType': pageType,
           'pageTitle': document.title,
           'timeThreshold': timeThreshold / 1000 // Convert to seconds for readability
         });
         
         // Set cookie to prevent showing again too soon
         document.cookie = cookieName + "=true; path=/; max-age=86400"; // 24 hours
       }, timeThreshold);
       
       // Clear timer if user leaves the page
       window.addEventListener('beforeunload', function() {
         if (window.timeOnPageTimer) {
           clearTimeout(window.timeOnPageTimer);
         }
       });
     })();
   </script>
   ```

7. Under **Triggering**, select **Window Loaded** trigger (create one if it doesn't exist).
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
     if (path.indexOf('/blog/') > -1 || path.indexOf('/news/') > -1) {
       return 'blog';
     } else if (path.indexOf('/product/') > -1 || path.indexOf('/item/') > -1) {
       return 'product';
     } else if (path.indexOf('/docs/') > -1 || path.indexOf('/help/') > -1) {
       return 'documentation';
     } else if (path.indexOf('/landing/') > -1 || document.querySelector('.landing-page')) {
       return 'landing';
     }
     
     // Check page elements as a fallback
     if (document.querySelector('article, .blog-post, .post')) {
       return 'blog';
     } else if (document.querySelector('.product-detail, .product-page')) {
       return 'product';
     } else if (document.querySelector('.documentation, .docs')) {
       return 'documentation';
     }
     
     // Default
     return 'general';
   }
   ```

7. Click **Save**.

---

## Step 4: Create Survey Display Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Time on Page Survey Display**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Get page information from dataLayer
       var pageType = {{dlv - pageType}} || 'page';
       var pageTitle = {{dlv - pageTitle}} || document.title;
       var timeSpent = {{dlv - timeThreshold}} || 60;
       
       // Create survey container
       var surveyContainer = document.createElement('div');
       surveyContainer.className = 'time-on-page-survey';
       surveyContainer.style.cssText = 'position:fixed; top:50%; left:50%; transform:translate(-50%, -50%); width:400px; background:#fff; border-radius:10px; box-shadow:0 0 25px rgba(0,0,0,0.3); padding:30px; z-index:999999; font-family:Arial,sans-serif;';
       
       // Create background overlay
       var overlay = document.createElement('div');
       overlay.style.cssText = 'position:fixed; top:0; left:0; width:100%; height:100%; background:rgba(0,0,0,0.6); z-index:999998;';
       
       // Customize survey based on page type
       var surveyQuestion = "What do you think about this page?";
       var surveyTitle = "We Value Your Feedback";
       
       if (pageType === 'blog') {
         surveyQuestion = "What do you think about this article?";
         surveyTitle = "Article Feedback";
       } else if (pageType === 'product') {
         surveyQuestion = "How would you rate this product information?";
         surveyTitle = "Product Page Feedback";
       } else if (pageType === 'documentation') {
         surveyQuestion = "Is this documentation helpful?";
         surveyTitle = "Documentation Feedback";
       } else if (pageType === 'landing') {
         surveyQuestion = "Did you find what you were looking for?";
         surveyTitle = "Quick Feedback";
       }
       
       // Survey content
       surveyContainer.innerHTML = `
         <div style="text-align:center; margin-bottom:25px;">
           <h2 style="margin:0 0 10px; color:#333; font-size:22px;">${surveyTitle}</h2>
           <p style="margin:0; color:#666; font-size:14px;">You've spent ${timeSpent} seconds on this page</p>
         </div>
         <p style="margin:0 0 20px; color:#444; font-size:16px; text-align:center;">${surveyQuestion}</p>
         <div style="display:flex; justify-content:center; gap:15px; margin-bottom:25px;">
           <button class="time-survey-rating" data-rating="positive" style="flex:1; padding:12px; background:#f0f0f0; border:none; border-radius:5px; cursor:pointer; font-size:15px; display:flex; flex-direction:column; align-items:center; max-width:100px;">
             <span style="font-size:24px; margin-bottom:5px;">👍</span>
             Helpful
           </button>
           <button class="time-survey-rating" data-rating="neutral" style="flex:1; padding:12px; background:#f0f0f0; border:none; border-radius:5px; cursor:pointer; font-size:15px; display:flex; flex-direction:column; align-items:center; max-width:100px;">
             <span style="font-size:24px; margin-bottom:5px;">😐</span>
             Neutral
           </button>
           <button class="time-survey-rating" data-rating="negative" style="flex:1; padding:12px; background:#f0f0f0; border:none; border-radius:5px; cursor:pointer; font-size:15px; display:flex; flex-direction:column; align-items:center; max-width:100px;">
             <span style="font-size:24px; margin-bottom:5px;">👎</span>
             Not Helpful
           </button>
         </div>
         <div id="timePageFeedback" style="display:none; margin-bottom:20px;">
           <textarea id="timePageFeedbackText" placeholder="Please tell us why..." style="width:100%; padding:12px; border:1px solid #ddd; border-radius:5px; resize:vertical; min-height:80px; box-sizing:border-box; font-family:inherit;"></textarea>
           <button id="submitTimeSurvey" style="width:100%; margin-top:15px; padding:12px; background:#4a90e2; color:#fff; border:none; border-radius:5px; cursor:pointer; font-weight:bold; font-size:16px;">Submit Feedback</button>
         </div>
         <div style="text-align:center;">
           <button id="closeTimeSurvey" style="background:none; border:none; color:#999; font-size:14px; cursor:pointer; text-decoration:underline;">No thanks</button>
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
       document.getElementById('closeTimeSurvey').addEventListener('click', closeSurvey);
       
       // Rating button functionality
       var ratingButtons = document.querySelectorAll('.time-survey-rating');
       var selectedRating = null;
       
       ratingButtons.forEach(function(button) {
         button.addEventListener('click', function() {
           // Get the rating value
           selectedRating = this.getAttribute('data-rating');
           
           // Reset all buttons
           ratingButtons.forEach(function(btn) {
             btn.style.background = '#f0f0f0';
             btn.style.boxShadow = 'none';
           });
           
           // Highlight selected button
           this.style.background = '#e0e7ff';
           this.style.boxShadow = '0 0 0 2px #4a90e2';
           
           // Show feedback text area for negative ratings
           if (selectedRating === 'negative') {
             document.getElementById('timePageFeedback').style.display = 'block';
           } else {
             // For positive/neutral ratings, submit immediately
             submitFeedback();
           }
         });
       });
       
       // Submit button functionality
       var submitButton = document.getElementById('submitTimeSurvey');
       if (submitButton) {
         submitButton.addEventListener('click', function() {
           submitFeedback();
         });
       }
       
       // Function to submit feedback
       function submitFeedback() {
         if (!selectedRating) return;
         
         var feedbackText = '';
         var feedbackTextArea = document.getElementById('timePageFeedbackText');
         if (feedbackTextArea && feedbackTextArea.value) {
           feedbackText = feedbackTextArea.value;
         }
         
         // Send the data
         dataLayer.push({
           'event': 'timeOnPageSurveyResponse',
           'pageType': pageType,
           'pageTitle': pageTitle,
           'timeThreshold': timeSpent,
           'surveyRating': selectedRating,
           'surveyFeedback': feedbackText
         });
         
         // Thank user
         surveyContainer.innerHTML = '<div style="text-align:center; padding:20px;"><h3 style="margin-bottom:15px; color:#333;">Thank You!</h3><p style="color:#666;">Your feedback helps us improve our content.</p></div>';
         
         // Remove after delay
         setTimeout(function() {
           closeSurvey();
         }, 3000);
       }
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Time on Page Survey Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `timeOnPageSurveyTrigger`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your survey tag, select **Time on Page Survey Trigger**.
9. Click **Save**.

---

## Step 5: Create Data Collection Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Time on Page Survey Response Collector**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code (replace with your actual survey endpoint):

   ```html
   <script>
     (function() {
       // Get response data from dataLayer
       var dlEvent = {{dataLayer}};
       
       // Only proceed if this is a time on page survey response
       if (dlEvent.event !== 'timeOnPageSurveyResponse') return;
       
       // Prepare data for sending
       var responseData = {
         type: 'timeOnPageSurvey',
         pageType: dlEvent.pageType,
         pageTitle: dlEvent.pageTitle,
         timeThreshold: dlEvent.timeThreshold,
         rating: dlEvent.surveyRating,
         feedback: dlEvent.surveyFeedback,
         url: window.location.href,
         timestamp: new Date().toISOString(),
         userId: {{User ID}} || 'anonymous'
       };
       
       // Send to your survey endpoint
       fetch('https://your-survey-endpoint.com/api/time-survey', {
         method: 'POST',
         headers: {
           'Content-Type': 'application/json',
         },
         body: JSON.stringify(responseData)
       })
       .then(response => response.json())
       .then(data => console.log('Time on page survey submitted:', data))
       .catch(error => console.error('Error submitting time on page survey:', error));
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Time on Page Survey Response Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `timeOnPageSurveyResponse`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your data collection tag, select **Time on Page Survey Response Trigger**.
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

   c. **dlv - timeThreshold**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `timeThreshold`

4. Click **Save** for each variable.

---

## Step 7: Create User Segments (Optional)

For more targeted surveys, you can create user segments to only show surveys to specific groups:

1. Go to the **Tags** section.
2. Edit your **Time on Page Timer** tag.
3. Modify the code to include user segmentation:

   ```html
   <script>
     (function() {
       // Existing code...
       
       // User segmentation check
       var showSurvey = true;
       
       // Example: Only show to returning visitors
       if ({{Returning Visitor}} !== true) {
         showSurvey = false;
       }
       
       // Example: Only show to certain traffic sources
       var trafficSource = {{Traffic Source}};
       if (trafficSource !== 'organic' && trafficSource !== 'referral') {
         showSurvey = false;
       }
       
       // If user doesn't match segments, exit
       if (!showSurvey) return;
       
       // Rest of the timer code...
     })();
   </script>
   ```

4. Click **Save**.

---

## Step 8: Test and Publish

1. Click the **Preview** button in GTM.
2. Visit one of your target pages.
3. Wait for the time threshold to be reached.
4. Verify that the survey appears.
5. Test each response option.
6. Check that additional feedback appears for negative ratings.
7. Verify data is being collected properly.
8. If everything functions correctly, publish your changes.

---

## Optional Enhancements

- **Progressive Time Thresholds:** Increase the time threshold for users who have already seen the survey, gathering feedback from more deeply engaged users.
- **Combined Triggers:** Use time on page in combination with scroll depth to ensure the user is actively engaged, not just leaving the page open.
- **Session-Based Limits:** Show the survey at most once per session, regardless of how many pages the user visits.
- **Engagement Qualification:** Only show the survey to users who have performed certain actions on your site (clicked specific elements, interacted with features).
- **A/B Testing Survey Timing:** Set up different time thresholds for different user groups to optimize when to ask for feedback.

---

## Advanced: Active Time on Page Tracking

For more accurate engagement measurement, you can track active time on page instead of just total time:

```html
<script>
  (function() {
    var activityTimeout;
    var isActive = true;
    var activeTime = 0;
    var activityCheckInterval = 1000; // Check every second
    var targetActiveTime = 60000; // 60 seconds of active time
    
    // Function to check for user activity
    function resetActivityTimeout() {
      clearTimeout(activityTimeout);
      isActive = true;
      
      // Reset after 5 seconds of inactivity
      activityTimeout = setTimeout(function() {
        isActive = false;
      }, 5000);
    }
    
    // Set up activity detection
    ['mousedown', 'mousemove', 'keypress', 'scroll', 'touchstart'].forEach(function(event) {
      document.addEventListener(event, resetActivityTimeout, true);
    });
    
    // Initially reset the timeout
    resetActivityTimeout();
    
    // Start tracking active time
    var trackingInterval = setInterval(function() {
      if (isActive) {
        activeTime += activityCheckInterval;
      }
      
      // If we've reached the target active time
      if (activeTime >= targetActiveTime) {
        clearInterval(trackingInterval);
        
        // Check if survey already shown
        var cookieName = 'activeTimeSurveyShown';
        if (document.cookie.indexOf(cookieName + '=') === -1) {
          // Trigger the survey
          dataLayer.push({
            'event': 'activeTimeOnPageTrigger',
            'activeTimeSeconds': activeTime / 1000,
            'pageType': {{Page Type}} || 'general'
          });
          
          // Set cookie to prevent showing again
          document.cookie = cookieName + "=true; path=/; max-age=86400"; // 24 hours
        }
      }
    }, activityCheckInterval);
  })();
</script>
```

This script tracks actual user engagement by monitoring activity and only counts time when the user is actively interacting with the page.

---
``` 