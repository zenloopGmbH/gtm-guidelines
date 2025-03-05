# GTM Scroll Depth Survey Implementation Guide

This guide explains how to implement scroll depth triggered surveys using Google Tag Manager (GTM) to gather feedback from users who have engaged with a significant portion of your content.

---

## Step 1: Determine Appropriate Scroll Depth Thresholds

Before implementation, decide which scroll depth thresholds should trigger the survey:

1. Common scroll depth thresholds:
   - 50% (halfway through content)
   - 75% (most of the content consumed)
   - 90% (nearly complete content consumption)
   - 100% (complete content consumption)

2. Consider different thresholds for different content types:
   - For blog articles: 75% might be appropriate
   - For product pages: 50% might be sufficient
   - For documentation: 90% might indicate serious interest

---

## Step 2: Create Scroll Depth Trigger

1. Navigate to **Triggers** in your GTM workspace.
2. Click the **New** button.
3. Name the trigger: **Scroll Depth Trigger - 75%**.
4. Click **Trigger Configuration**.
5. Select **Scroll Depth**.
6. Configure the settings:
   - **Vertical Scroll Depths**: Select **Percentages**
   - Enter the percentage value: `75` (or your chosen threshold)
   - Check **Enable advanced settings and check conditions**
   - **This trigger fires on**: **Some Pages**
   - Set up conditions for your target pages (e.g., **Page Path** contains `/blog/`)
7. Click **Save**.

---

## Step 3: Create Scroll Depth Event Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Scroll Depth Event Tracker**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Check if scroll survey already shown in this session
       var sessionStorageKey = 'scrollSurveyShown';
       var cookieName = 'scrollSurveyShown';
       
       // Don't show if already shown in this session
       if (sessionStorage.getItem(sessionStorageKey) === 'true') return;
       
       // Don't show if already shown in the last 7 days
       if (document.cookie.indexOf(cookieName + '=') !== -1) return;
       
       // Store content metadata
       var contentData = {
         title: document.title,
         url: window.location.href,
         path: window.location.pathname,
         scrollPercentage: {{dlv - Scroll Depth Threshold}} || '75',
         contentType: {{Page Type}} || 'article'
       };
       
       // Store this data for use in the survey
       window.scrollContentData = contentData;
       
       // Trigger the survey after a short delay (1.5 seconds)
       setTimeout(function() {
         dataLayer.push({
           'event': 'scrollDepthSurveyTrigger',
           'contentTitle': contentData.title,
           'contentType': contentData.contentType,
           'scrollPercentage': contentData.scrollPercentage
         });
         
         // Mark as shown in this session
         sessionStorage.setItem(sessionStorageKey, 'true');
         
         // Set cookie to prevent showing again too soon
         document.cookie = cookieName + "=true; path=/; max-age=604800"; // 7 days
       }, 1500);
     })();
   </script>
   ```

7. Under **Triggering**, select the **Scroll Depth Trigger** you created.
8. Click **Save**.

---

## Step 4: Create Survey Display Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Scroll Depth Survey Display**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Get content information from dataLayer or window object
       var contentTitle = {{dlv - contentTitle}} || document.title;
       var contentType = {{dlv - contentType}} || 'content';
       var scrollPercentage = {{dlv - scrollPercentage}} || '75%';
       
       // Create survey container
       var surveyContainer = document.createElement('div');
       surveyContainer.className = 'scroll-feedback-survey';
       surveyContainer.style.cssText = 'position:fixed; bottom:30px; right:30px; width:360px; background:#fff; border-radius:10px; box-shadow:0 0 20px rgba(0,0,0,0.2); padding:25px; z-index:999999; font-family:Arial,sans-serif; transition:all 0.3s ease-in-out;';
       
       // Survey content - customize based on content type
       var surveyQuestion = "How would you rate this content?";
       var surveyTitle = "Content Feedback";
       
       if (contentType === 'blog' || contentType === 'article') {
         surveyQuestion = "How helpful was this article?";
         surveyTitle = "Article Feedback";
       } else if (contentType === 'product') {
         surveyQuestion = "Is this product information helpful?";
         surveyTitle = "Product Feedback";
       } else if (contentType === 'documentation') {
         surveyQuestion = "Did this documentation answer your questions?";
         surveyTitle = "Documentation Feedback";
       }
       
       // Survey content
       surveyContainer.innerHTML = `
         <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:15px;">
           <h3 style="margin:0; color:#333; font-size:18px;">${surveyTitle}</h3>
           <button id="closeScrollSurvey" style="background:none; border:none; cursor:pointer; font-size:22px; color:#999;">&times;</button>
         </div>
         <p style="margin:0 0 15px; color:#666; font-size:15px;">${surveyQuestion}</p>
         <div style="display:flex; flex-direction:column; gap:10px; margin-bottom:20px;">
           <div class="survey-option" data-value="very-helpful" style="padding:10px; border:1px solid #ddd; border-radius:5px; cursor:pointer;">
             Very helpful
           </div>
           <div class="survey-option" data-value="somewhat-helpful" style="padding:10px; border:1px solid #ddd; border-radius:5px; cursor:pointer;">
             Somewhat helpful
           </div>
           <div class="survey-option" data-value="not-helpful" style="padding:10px; border:1px solid #ddd; border-radius:5px; cursor:pointer;">
             Not helpful
           </div>
           <div class="survey-option" data-value="looking-for-something-else" style="padding:10px; border:1px solid #ddd; border-radius:5px; cursor:pointer;">
             I was looking for something else
           </div>
         </div>
         <div id="additionalFeedback" style="display:none; margin-bottom:15px;">
           <textarea id="scrollFeedbackText" placeholder="Please tell us more about your feedback..." style="width:100%; padding:10px; border:1px solid #ddd; border-radius:5px; resize:vertical; min-height:80px; box-sizing:border-box;"></textarea>
           <button id="submitScrollSurvey" style="width:100%; margin-top:10px; padding:10px; background:#4a90e2; color:#fff; border:none; border-radius:5px; cursor:pointer; font-weight:bold;">Submit Feedback</button>
         </div>
       `;
       
       // Add to page
       document.body.appendChild(surveyContainer);
       
       // Animate in
       setTimeout(function() {
         surveyContainer.style.transform = 'translateY(0)';
       }, 100);
       
       // Close button functionality
       document.getElementById('closeScrollSurvey').addEventListener('click', function() {
         document.body.removeChild(surveyContainer);
       });
       
       // Make survey options clickable
       var surveyOptions = document.querySelectorAll('.survey-option');
       var selectedOption = null;
       
       surveyOptions.forEach(function(option) {
         option.addEventListener('click', function() {
           // Reset all options
           surveyOptions.forEach(function(opt) {
             opt.style.backgroundColor = '#fff';
             opt.style.borderColor = '#ddd';
             opt.style.color = '#333';
           });
           
           // Highlight selected option
           this.style.backgroundColor = '#4a90e2';
           this.style.borderColor = '#4a90e2';
           this.style.color = '#fff';
           
           // Store selected value
           selectedOption = this.getAttribute('data-value');
           
           // Show additional feedback for negative responses
           if (selectedOption === 'not-helpful' || selectedOption === 'looking-for-something-else') {
             document.getElementById('additionalFeedback').style.display = 'block';
           } else {
             // For positive feedback, submit immediately
             submitFeedback();
           }
         });
       });
       
       // Submit button functionality
       var submitButton = document.getElementById('submitScrollSurvey');
       if (submitButton) {
         submitButton.addEventListener('click', function() {
           submitFeedback();
         });
       }
       
       // Function to submit feedback
       function submitFeedback() {
         if (!selectedOption) return;
         
         var feedbackText = '';
         var feedbackTextArea = document.getElementById('scrollFeedbackText');
         if (feedbackTextArea) {
           feedbackText = feedbackTextArea.value || '';
         }
         
         // Send the data
         dataLayer.push({
           'event': 'scrollDepthSurveyResponse',
           'contentTitle': contentTitle,
           'contentType': contentType,
           'scrollPercentage': scrollPercentage,
           'responseValue': selectedOption,
           'responseFeedback': feedbackText
         });
         
         // Thank user
         surveyContainer.innerHTML = '<div style="text-align:center; padding:20px;"><h3 style="margin-bottom:10px;">Thank You!</h3><p>Your feedback helps us create better content.</p></div>';
         
         // Remove after delay
         setTimeout(function() {
           if (document.body.contains(surveyContainer)) {
             surveyContainer.style.opacity = '0';
             setTimeout(function() {
               document.body.removeChild(surveyContainer);
             }, 300);
           }
         }, 3000);
       }
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Scroll Depth Survey Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `scrollDepthSurveyTrigger`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your survey tag, select **Scroll Depth Survey Trigger**.
9. Click **Save**.

---

## Step 5: Create Data Collection Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Scroll Depth Survey Response Collector**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code (replace with your actual survey endpoint):

   ```html
   <script>
     (function() {
       // Get response data from dataLayer
       var dlEvent = {{dataLayer}};
       
       // Only proceed if this is a scroll depth survey response
       if (dlEvent.event !== 'scrollDepthSurveyResponse') return;
       
       // Prepare data for sending
       var responseData = {
         type: 'scrollDepthSurvey',
         contentTitle: dlEvent.contentTitle,
         contentType: dlEvent.contentType,
         scrollPercentage: dlEvent.scrollPercentage,
         responseValue: dlEvent.responseValue,
         responseFeedback: dlEvent.responseFeedback,
         url: window.location.href,
         timestamp: new Date().toISOString(),
         userId: {{User ID}} || 'anonymous'
       };
       
       // Send to your survey endpoint
       fetch('https://your-survey-endpoint.com/api/scroll-survey', {
         method: 'POST',
         headers: {
           'Content-Type': 'application/json',
         },
         body: JSON.stringify(responseData)
       })
       .then(response => response.json())
       .then(data => console.log('Scroll survey submitted:', data))
       .catch(error => console.error('Error submitting scroll survey:', error));
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Scroll Depth Survey Response Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `scrollDepthSurveyResponse`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your data collection tag, select **Scroll Depth Survey Response Trigger**.
9. Click **Save**.

---

## Step 6: Variable Setup for DataLayer Values

1. Go to **Variables** in your GTM workspace.
2. Click the **New** button under "User-Defined Variables".
3. Create the following variables:

   a. **dlv - contentTitle**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `contentTitle`

   b. **dlv - contentType**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `contentType`

   c. **dlv - scrollPercentage**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `scrollPercentage`
   
   d. **dlv - Scroll Depth Threshold**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `Scroll Depth Threshold`

4. For the Page Type variable:
   - Create a new variable named **Page Type**
   - Choose **Custom JavaScript** as the variable type
   - Enter the following code (customize based on your site's structure):

   ```javascript
   function() {
     var path = window.location.pathname;
     if (path.indexOf('/blog/') > -1 || path.indexOf('/article/') > -1) {
       return 'blog';
     } else if (path.indexOf('/product/') > -1) {
       return 'product';
     } else if (path.indexOf('/docs/') > -1 || path.indexOf('/documentation/') > -1) {
       return 'documentation';
     } else {
       return 'general';
     }
   }
   ```

5. Click **Save** for each variable.

---

## Step 7: Test and Publish

1. Click the **Preview** button in GTM.
2. Visit one of your target pages.
3. Scroll down to trigger the scroll depth threshold.
4. Verify that the survey appears.
5. Test each response option.
6. Check that additional feedback appears for negative responses.
7. Verify data is being collected properly.
8. If everything functions correctly, publish your changes.

---

## Optional Enhancements

- **Multiple Scroll Thresholds:** Create different triggers for different scroll depths to gather insights at various engagement levels.
- **Content Type Specific Surveys:** Customize the survey questions based on the type of content being viewed.
- **Combine with Time on Page:** Only trigger the survey when both scroll depth and time on page thresholds are met.
- **Segmented Deployment:** Show the survey only to specific user segments based on their behavior or demographics.
- **Adaptive Timing:** Adjust when the survey appears based on the length of the content (longer content might need a higher scroll percentage).

---

## Example: Detecting Content Type Automatically

You can enhance the content type detection by adding more sophisticated logic:

```javascript
function detectContentType() {
  // Check for schema markup
  var schemaType = document.querySelector('meta[property="og:type"]');
  if (schemaType) {
    var type = schemaType.getAttribute('content');
    if (type === 'article') return 'blog';
    if (type.includes('product')) return 'product';
  }
  
  // Check URL patterns
  var path = window.location.pathname;
  if (path.match(/\/(blog|news|article)\//) || document.querySelector('article')) {
    return 'blog';
  }
  
  if (path.match(/\/(product|item)\//) || document.querySelector('.product-page, .product-detail')) {
    return 'product';
  }
  
  if (path.match(/\/(docs|documentation|help|guide|manual)\//) || document.querySelector('.documentation')) {
    return 'documentation';
  }
  
  // Check page content indicators
  if (document.querySelector('.blog-post, .article, .post')) {
    return 'blog';
  }
  
  // Default
  return 'general';
}

// Use this function in your GTM setup
```

---