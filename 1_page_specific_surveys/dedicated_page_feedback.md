# GTM Dedicated Page Feedback Implementation Guide

This guide explains how to implement a dedicated page feedback survey using Google Tag Manager (GTM) to collect targeted insights about specific pages on your website.

---

## Step 1: Identify Target Pages

Before implementation, decide which specific pages should trigger the survey:
- Product pages
- Landing pages
- Checkout process pages
- Thank you/confirmation pages

---

## Step 2: Create Page View Trigger

1. Navigate to **Triggers** in your GTM workspace.
2. Click the **New** button.
3. Name the trigger: **Dedicated Page View Trigger**.
4. Click **Trigger Configuration**.
5. Select **Page View**.
6. Choose **Some Page Views**.
7. Set up conditions for your specific pages:
   - **Page Path** contains `/product/` (for product pages)
   - OR **Page Path** contains `/landing/` (for landing pages)
   - OR **Page Path** equals `/checkout/` (for checkout page)
   - (Add additional conditions for any other specific pages)
8. Click **Save**.

---

## Step 3: Create Survey Delay Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Page Survey Delay**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Only set timeout once
       if (window.pageDelayStarted) return;
       window.pageDelayStarted = true;
       
       // Wait for 10 seconds before triggering the survey
       // Adjust time as needed for your use case
       setTimeout(function() {
         // Check if survey already shown
         if (document.cookie.indexOf('pageSurveyShown=') === -1) {
           // Fire the custom event after delay
           dataLayer.push({'event': 'showPageSurvey'});
           // Set cookie to prevent multiple triggers
           document.cookie = "pageSurveyShown=true; path=/; max-age=86400"; // 24 hours
         }
       }, 10000); // 10 seconds delay
     })();
   </script>
   ```

7. Under **Triggering**, select your **Dedicated Page View Trigger**.
8. Click **Save**.

---

## Step 4: Create Survey Display Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Page Survey Display**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Create survey container
       var surveyContainer = document.createElement('div');
       surveyContainer.className = 'page-feedback-survey';
       surveyContainer.style.cssText = 'position:fixed; bottom:20px; right:20px; width:350px; background:#fff; border-radius:10px; box-shadow:0 0 15px rgba(0,0,0,0.2); padding:20px; z-index:999999; font-family:Arial,sans-serif;';
       
       // Survey content
       surveyContainer.innerHTML = `
         <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:15px;">
           <h3 style="margin:0; color:#333; font-size:18px;">Page Feedback</h3>
           <button id="closeSurvey" style="background:none; border:none; cursor:pointer; font-size:20px; color:#999;">&times;</button>
         </div>
         <p style="margin:0 0 15px; color:#666; font-size:14px;">How would you rate your experience with this page?</p>
         <div style="display:flex; justify-content:space-between; margin-bottom:15px;">
           <button class="rating-btn" data-rating="1" style="flex:1; margin:0 5px; padding:8px; border:1px solid #ddd; background:#f9f9f9; border-radius:5px; cursor:pointer;">1</button>
           <button class="rating-btn" data-rating="2" style="flex:1; margin:0 5px; padding:8px; border:1px solid #ddd; background:#f9f9f9; border-radius:5px; cursor:pointer;">2</button>
           <button class="rating-btn" data-rating="3" style="flex:1; margin:0 5px; padding:8px; border:1px solid #ddd; background:#f9f9f9; border-radius:5px; cursor:pointer;">3</button>
           <button class="rating-btn" data-rating="4" style="flex:1; margin:0 5px; padding:8px; border:1px solid #ddd; background:#f9f9f9; border-radius:5px; cursor:pointer;">4</button>
           <button class="rating-btn" data-rating="5" style="flex:1; margin:0 5px; padding:8px; border:1px solid #ddd; background:#f9f9f9; border-radius:5px; cursor:pointer;">5</button>
         </div>
         <textarea id="surveyFeedback" placeholder="Please share any additional feedback..." style="width:100%; padding:10px; border:1px solid #ddd; border-radius:5px; resize:vertical; min-height:80px; box-sizing:border-box; margin-bottom:15px;"></textarea>
         <button id="submitSurvey" style="width:100%; padding:10px; background:#4a90e2; color:#fff; border:none; border-radius:5px; cursor:pointer; font-weight:bold;">Submit Feedback</button>
       `;
       
       // Add to page
       document.body.appendChild(surveyContainer);
       
       // Close button functionality
       document.getElementById('closeSurvey').addEventListener('click', function() {
         document.body.removeChild(surveyContainer);
       });
       
       // Submit functionality
       document.getElementById('submitSurvey').addEventListener('click', function() {
         var selectedRating = document.querySelector('.rating-btn.selected');
         var feedbackText = document.getElementById('surveyFeedback').value;
         
         if (selectedRating) {
           // Send data to your analytics or to Zenloop
           dataLayer.push({
             'event': 'surveySubmitted',
             'surveyType': 'pageFeedback',
             'surveyRating': selectedRating.getAttribute('data-rating'),
             'surveyFeedback': feedbackText,
             'surveyPage': window.location.pathname
           });
           
           // Thank user
           surveyContainer.innerHTML = '<div style="text-align:center; padding:20px;"><h3>Thank You!</h3><p>Your feedback helps us improve.</p></div>';
           
           // Remove after delay
           setTimeout(function() {
             if (document.body.contains(surveyContainer)) {
               document.body.removeChild(surveyContainer);
             }
           }, 3000);
         } else {
           alert('Please select a rating');
         }
       });
       
       // Rating button selection
       var ratingButtons = document.querySelectorAll('.rating-btn');
       ratingButtons.forEach(function(button) {
         button.addEventListener('click', function() {
           // Remove selected class from all buttons
           ratingButtons.forEach(function(btn) {
             btn.classList.remove('selected');
             btn.style.background = '#f9f9f9';
             btn.style.color = '#333';
           });
           
           // Add selected class to clicked button
           this.classList.add('selected');
           this.style.background = '#4a90e2';
           this.style.color = '#fff';
         });
       });
     })();
   </script>
   ```

7. Create a new trigger for this tag:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Show Page Survey Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `showPageSurvey`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your survey tag, select **Show Page Survey Trigger**.
9. Click **Save**.

---

## Step 5: Create Data Collection Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Survey Data Collection**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code (replace with your actual survey endpoint):

   ```html
   <script>
     (function() {
       // Get survey data from dataLayer
       var dlEvent = {{dataLayer}};
       
       // Only proceed if this is a survey submission
       if (dlEvent.event !== 'surveySubmitted') return;
       
       // Prepare data for sending
       var surveyData = {
         type: dlEvent.surveyType,
         rating: dlEvent.surveyRating,
         feedback: dlEvent.surveyFeedback,
         page: dlEvent.surveyPage,
         timestamp: new Date().toISOString(),
         userId: {{User ID}} || 'anonymous'
       };
       
       // Send to your survey endpoint
       fetch('https://your-survey-endpoint.com/api/submit', {
         method: 'POST',
         headers: {
           'Content-Type': 'application/json',
         },
         body: JSON.stringify(surveyData)
       })
       .then(response => response.json())
       .then(data => console.log('Survey submitted:', data))
       .catch(error => console.error('Error submitting survey:', error));
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Survey Submitted Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `surveySubmitted`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your data collection tag, select **Survey Submitted Trigger**.
9. Click **Save**.

---

## Step 6: Test and Publish

1. Click the **Preview** button in GTM.
2. Visit one of your target pages.
3. Verify that after the delay, the survey appears.
4. Test submitting feedback and verify data is being collected properly.
5. If everything functions correctly, publish your changes.

---

## Optional Enhancements

- **Multiple Page Types:** Create different survey questions for different page types by setting up multiple tags with different triggers.
- **Frequency Capping:** Modify the cookie duration to control how often a user sees surveys.
- **Targeting Segments:** Add more conditions to the trigger to target specific user segments.
- **A/B Testing:** Create variations of your survey to test different approaches.

--- 