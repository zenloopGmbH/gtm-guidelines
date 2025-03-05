# GTM Section-Based Feedback Implementation Guide

This guide explains how to implement section-based feedback surveys using Google Tag Manager (GTM) to gather insights about specific sections of your website, such as blog articles, help center pages, or product categories.

---

## Step 1: Identify and Mark Target Sections

First, you need to mark the sections on your website where you want to collect feedback:

1. Add unique data attributes to the HTML elements containing your target sections:

   ```html
   <!-- Example for a blog article section -->
   <section id="blogContent" data-survey-section="blog-content">
     <!-- Your blog content here -->
   </section>

   <!-- Example for a product documentation section -->
   <div id="productDocumentation" data-survey-section="product-docs">
     <!-- Your documentation content here -->
   </div>
   ```

2. Alternatively, if you can't modify the HTML directly, identify unique CSS selectors for each section.

---

## Step 2: Create Section Visibility Trigger

1. Navigate to **Triggers** in your GTM workspace.
2. Click the **New** button.
3. Name the trigger: **Section Visibility Trigger**.
4. Click **Trigger Configuration**.
5. Select **Element Visibility**.
6. Configure the trigger with these settings:
   - **Selection Method**: CSS Selector
   - **Element Selector**: `[data-survey-section]` (or your specific selectors)
   - **When to fire this trigger**: 
     - Once per page
     - When element is visible in viewport
     - Minimum Percent Visible: 50%
   - **Advanced Settings**:
     - Observe DOM changes: Yes
7. Click **Save**.

---

## Step 3: Create Section Data Collection Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Section Data Collector**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Get the section element that triggered this
       var element = {{Element}};
       if (!element) return;
       
       // Get section identifier and check if we've already shown a survey for this section
       var sectionId = element.getAttribute('data-survey-section');
       var cookieName = 'sectionSurvey_' + sectionId;
       
       if (document.cookie.indexOf(cookieName + '=') !== -1) return;
       
       // Store section data for later use
       window.currentSectionId = sectionId;
       window.currentSectionTitle = element.getAttribute('data-survey-title') || document.title;
       
       // Create a delayed trigger for the survey (30 seconds after viewing section)
       setTimeout(function() {
         dataLayer.push({
           'event': 'sectionSurveyTrigger',
           'sectionId': sectionId,
           'sectionTitle': window.currentSectionTitle
         });
         
         // Set cookie to prevent showing again
         document.cookie = cookieName + "=true; path=/; max-age=604800"; // 7 days
       }, 30000); // 30 seconds delay
     })();
   </script>
   ```

7. Under **Triggering**, select your **Section Visibility Trigger**.
8. Click **Save**.

---

## Step 4: Create Section Survey Display Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Section Survey Display**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Get section details from dataLayer
       var sectionId = {{dlv - sectionId}};
       var sectionTitle = {{dlv - sectionTitle}} || 'this section';
       
       // Create survey container
       var surveyContainer = document.createElement('div');
       surveyContainer.className = 'section-feedback-survey';
       surveyContainer.style.cssText = 'position:fixed; bottom:20px; right:20px; width:350px; background:#fff; border-radius:10px; box-shadow:0 0 15px rgba(0,0,0,0.2); padding:20px; z-index:999999; font-family:Arial,sans-serif;';
       
       // Survey content
       surveyContainer.innerHTML = `
         <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:15px;">
           <h3 style="margin:0; color:#333; font-size:18px;">Section Feedback</h3>
           <button id="closeSectionSurvey" style="background:none; border:none; cursor:pointer; font-size:20px; color:#999;">&times;</button>
         </div>
         <p style="margin:0 0 15px; color:#666; font-size:14px;">Was this section helpful?</p>
         <div style="display:flex; justify-content:space-between; margin-bottom:15px;">
           <button id="sectionYes" style="flex:1; margin:0 5px; padding:8px; border:1px solid #ddd; background:#f9f9f9; border-radius:5px; cursor:pointer;">Yes</button>
           <button id="sectionNo" style="flex:1; margin:0 5px; padding:8px; border:1px solid #ddd; background:#f9f9f9; border-radius:5px; cursor:pointer;">No</button>
         </div>
         <div id="additionalFeedback" style="display:none;">
           <textarea id="sectionFeedback" placeholder="How could we improve this section?" style="width:100%; padding:10px; border:1px solid #ddd; border-radius:5px; resize:vertical; min-height:80px; box-sizing:border-box; margin-bottom:15px;"></textarea>
           <button id="submitSectionSurvey" style="width:100%; padding:10px; background:#4a90e2; color:#fff; border:none; border-radius:5px; cursor:pointer; font-weight:bold;">Submit Feedback</button>
         </div>
       `;
       
       // Add to page
       document.body.appendChild(surveyContainer);
       
       // Close button functionality
       document.getElementById('closeSectionSurvey').addEventListener('click', function() {
         document.body.removeChild(surveyContainer);
       });
       
       // Yes/No button functionality
       document.getElementById('sectionYes').addEventListener('click', function() {
         // Record the positive response
         dataLayer.push({
           'event': 'sectionSurveyResponse',
           'sectionId': sectionId,
           'sectionTitle': sectionTitle,
           'response': 'yes',
           'feedbackText': ''
         });
         
         // Thank user
         surveyContainer.innerHTML = '<div style="text-align:center; padding:20px;"><h3>Thank You!</h3><p>We\'re glad this was helpful.</p></div>';
         
         // Remove after delay
         setTimeout(function() {
           if (document.body.contains(surveyContainer)) {
             document.body.removeChild(surveyContainer);
           }
         }, 3000);
       });
       
       document.getElementById('sectionNo').addEventListener('click', function() {
         // Show additional feedback field
         document.getElementById('additionalFeedback').style.display = 'block';
         
         // Style the No button as selected
         this.style.background = '#f0f0f0';
         this.style.borderColor = '#999';
         
         // Reset Yes button
         document.getElementById('sectionYes').style.background = '#f9f9f9';
         document.getElementById('sectionYes').style.borderColor = '#ddd';
       });
       
       // Submit functionality
       document.getElementById('submitSectionSurvey').addEventListener('click', function() {
         var feedbackText = document.getElementById('sectionFeedback').value;
         
         // Record the negative response with feedback
         dataLayer.push({
           'event': 'sectionSurveyResponse',
           'sectionId': sectionId,
           'sectionTitle': sectionTitle,
           'response': 'no',
           'feedbackText': feedbackText
         });
         
         // Thank user
         surveyContainer.innerHTML = '<div style="text-align:center; padding:20px;"><h3>Thank You!</h3><p>Your feedback helps us improve.</p></div>';
         
         // Remove after delay
         setTimeout(function() {
           if (document.body.contains(surveyContainer)) {
             document.body.removeChild(surveyContainer);
           }
         }, 3000);
       });
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Section Survey Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `sectionSurveyTrigger`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your survey tag, select **Section Survey Trigger**.
9. Click **Save**.

---

## Step 5: Create Data Collection Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Section Survey Response Collector**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code (replace with your actual survey endpoint):

   ```html
   <script>
     (function() {
       // Get response data from dataLayer
       var dlEvent = {{dataLayer}};
       
       // Only proceed if this is a survey response
       if (dlEvent.event !== 'sectionSurveyResponse') return;
       
       // Prepare data for sending
       var responseData = {
         type: 'sectionFeedback',
         sectionId: dlEvent.sectionId,
         sectionTitle: dlEvent.sectionTitle,
         response: dlEvent.response,
         feedback: dlEvent.feedbackText,
         url: window.location.href,
         timestamp: new Date().toISOString(),
         userId: {{User ID}} || 'anonymous'
       };
       
       // Send to your survey endpoint
       fetch('https://your-survey-endpoint.com/api/section-feedback', {
         method: 'POST',
         headers: {
           'Content-Type': 'application/json',
         },
         body: JSON.stringify(responseData)
       })
       .then(response => response.json())
       .then(data => console.log('Section feedback submitted:', data))
       .catch(error => console.error('Error submitting section feedback:', error));
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Section Survey Response Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `sectionSurveyResponse`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your data collection tag, select **Section Survey Response Trigger**.
9. Click **Save**.

---

## Step 6: Variable Setup for DataLayer Values

1. Go to **Variables** in your GTM workspace.
2. Click the **New** button under "User-Defined Variables".
3. Name the variable: **dlv - sectionId**.
4. Click **Variable Configuration**.
5. Select **Data Layer Variable**.
6. Set **Data Layer Variable Name** to `sectionId`.
7. Click **Save**.

8. Repeat steps 2-7 to create a second variable:
   - Name: **dlv - sectionTitle**
   - Type: Data Layer Variable
   - Data Layer Variable Name: `sectionTitle`

---

## Step 7: Test and Publish

1. Click the **Preview** button in GTM.
2. Visit a page with your marked sections.
3. Scroll to one of the sections to make it visible.
4. Wait for the configured delay time.
5. Verify that the survey appears.
6. Test both positive and negative responses.
7. Check that data is being collected properly.
8. If everything functions correctly, publish your changes.

---

## Optional Enhancements

- **Different Surveys by Section Type:** Create different survey questions for different section types by using the sectionId to customize the survey content.
- **Personalization:** Add user context to tailor the survey to different user segments.
- **Follow-up Questions:** Add conditional follow-up questions based on initial responses.
- **Survey Rotation:** Rotate different types of questions to get varied feedback about sections.
- **Read-depth Tracking:** Combine with scroll tracking to ensure users have consumed a meaningful amount of the section before asking for feedback.

--- 