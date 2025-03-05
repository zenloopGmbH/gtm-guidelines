# GTM Custom Events Survey Implementation Guide

This guide explains how to implement surveys triggered by custom events using Google Tag Manager (GTM) to gather feedback based on unique user interactions like form submissions, video plays, or engagement with interactive elements.

---

## Step 1: Identify Custom Events to Track

First, identify the custom events that are meaningful for your website:

1. Common custom events to consider:
   - Form submissions
   - Video plays, pauses, or completions
   - Interactive widget engagement
   - Product configurator interactions
   - Search queries
   - Modal/popup interactions
   - Custom calculator usage
   - Filtering or sorting products
   - Account interactions (not tied to page loads)

2. Check if these events are already being pushed to the dataLayer or need to be implemented.

---

## Step 2: Implement Custom Event Tracking

If your custom events aren't already being tracked, you'll need to implement them first:

### Option A: If You Control the Website Code

Add dataLayer push events at appropriate points in your code:

```javascript
// Example: Track form submission
document.getElementById('contactForm').addEventListener('submit', function() {
  dataLayer.push({
    'event': 'formSubmission',
    'formId': 'contactForm',
    'formName': 'Contact Us'
  });
});

// Example: Track video interactions
document.getElementById('productVideo').addEventListener('play', function() {
  dataLayer.push({
    'event': 'videoPlay',
    'videoId': 'productVideo',
    'videoName': 'Product Demo'
  });
});
```

### Option B: Using GTM to Create Custom Events

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Form Submission Tracker**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code (customize for your form selectors):

   ```html
   <script>
     (function() {
       // Find all forms you want to track
       var forms = document.querySelectorAll('form.track-this, #contactForm, #signupForm');
       
       forms.forEach(function(form) {
         form.addEventListener('submit', function() {
           // Get form details
           var formId = this.id || 'unnamed-form';
           var formName = this.getAttribute('data-form-name') || formId;
           
           // Push to dataLayer
           dataLayer.push({
             'event': 'formSubmission',
             'formId': formId,
             'formName': formName
           });
         });
       });
     })();
   </script>
   ```

7. Under **Triggering**, select **All Pages** or a more specific trigger.
8. Click **Save**.

---

## Step 3: Create Custom Event Trigger

1. Navigate to **Triggers** in your GTM workspace.
2. Click the **New** button.
3. Name the trigger: **Custom Event Trigger - Form Submission**.
4. Click **Trigger Configuration**.
5. Select **Custom Event**.
6. For **Event Name**, enter: `formSubmission` (or your specific custom event name).
7. Choose **Some Custom Events** if you want to add conditions.
8. Add conditions if needed (e.g., **formId equals** `contactForm`).
9. Click **Save**.

---

## Step 4: Create Survey Delay Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Custom Event Survey Delay**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Get custom event details from the dataLayer
       var eventData = {
         eventType: {{Event}},
         formId: {{dlv - formId}} || '',
         formName: {{dlv - formName}} || '',
         videoId: {{dlv - videoId}} || '',
         videoName: {{dlv - videoName}} || ''
       };
       
       // Store event type for cookie naming
       var eventType = eventData.eventType;
       var eventName = eventData.formName || eventData.videoName || eventType;
       
       // Create a unique identifier for this event
       var eventIdentifier = eventType + '_' + (eventData.formId || eventData.videoId || 'general');
       var cookieName = 'eventSurvey_' + eventIdentifier.replace(/\s+/g, '_').toLowerCase();
       
       // Check if we've already shown a survey for this event type
       if (document.cookie.indexOf(cookieName + '=') !== -1) return;
       
       // Store event data for later use
       window.customEventData = eventData;
       
       // Set appropriate delay based on event type
       var surveyDelay = 2000; // Default 2 seconds
       
       if (eventType === 'formSubmission') {
         surveyDelay = 3000; // Show 3 seconds after form submission
       } else if (eventType === 'videoPlay') {
         surveyDelay = 10000; // Show 10 seconds after video starts
       } else if (eventType === 'videoComplete') {
         surveyDelay = 1000; // Show 1 second after video completes
       }
       
       // Trigger the survey after the specified delay
       setTimeout(function() {
         dataLayer.push({
           'event': 'customEventSurveyTrigger',
           'eventType': eventType,
           'eventName': eventName,
           'eventIdentifier': eventIdentifier
         });
         
         // Set cookie to prevent showing again
         document.cookie = cookieName + "=true; path=/; max-age=604800"; // 7 days
       }, surveyDelay);
     })();
   </script>
   ```

7. Under **Triggering**, select your **Custom Event Trigger** created in Step 3.
8. Click **Save**.

---

## Step 5: Create Survey Display Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Custom Event Survey Display**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code:

   ```html
   <script>
     (function() {
       // Get event details from dataLayer
       var eventType = {{dlv - eventType}} || 'interaction';
       var eventName = {{dlv - eventName}} || 'this feature';
       
       // Create survey container
       var surveyContainer = document.createElement('div');
       surveyContainer.className = 'custom-event-survey';
       surveyContainer.style.cssText = 'position:fixed; bottom:20px; right:20px; width:380px; background:#fff; border-radius:10px; box-shadow:0 0 15px rgba(0,0,0,0.2); padding:20px; z-index:999999; font-family:Arial,sans-serif;';
       
       // Customize question based on the event type
       var surveyTitle = 'Feedback';
       var questionText = 'How was your experience with ' + eventName + '?';
       
       if (eventType === 'formSubmission') {
         surveyTitle = 'Form Feedback';
         questionText = 'How was your experience with our ' + eventName + ' form?';
       } else if (eventType.includes('video')) {
         surveyTitle = 'Video Feedback';
         questionText = 'How would you rate the ' + eventName + ' video?';
       } else if (eventType.includes('search')) {
         surveyTitle = 'Search Feedback';
         questionText = 'Did you find what you were looking for?';
       }
       
       // Survey content
       surveyContainer.innerHTML = `
         <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:15px;">
           <h3 style="margin:0; color:#333; font-size:18px;">${surveyTitle}</h3>
           <button id="closeEventSurvey" style="background:none; border:none; cursor:pointer; font-size:20px; color:#999;">&times;</button>
         </div>
         <p style="margin:0 0 15px; color:#666; font-size:14px;">${questionText}</p>
         <div style="display:flex; justify-content:space-between; margin-bottom:15px;">
           <button class="rating-btn" data-rating="1" style="flex:1; margin:0 5px; padding:8px; border:1px solid #ddd; background:#f9f9f9; border-radius:5px; cursor:pointer;">1</button>
           <button class="rating-btn" data-rating="2" style="flex:1; margin:0 5px; padding:8px; border:1px solid #ddd; background:#f9f9f9; border-radius:5px; cursor:pointer;">2</button>
           <button class="rating-btn" data-rating="3" style="flex:1; margin:0 5px; padding:8px; border:1px solid #ddd; background:#f9f9f9; border-radius:5px; cursor:pointer;">3</button>
           <button class="rating-btn" data-rating="4" style="flex:1; margin:0 5px; padding:8px; border:1px solid #ddd; background:#f9f9f9; border-radius:5px; cursor:pointer;">4</button>
           <button class="rating-btn" data-rating="5" style="flex:1; margin:0 5px; padding:8px; border:1px solid #ddd; background:#f9f9f9; border-radius:5px; cursor:pointer;">5</button>
         </div>
         <div id="ratingFeedback" style="display:none; margin-bottom:15px;">
           <textarea id="eventFeedback" placeholder="Please tell us more about your experience..." style="width:100%; padding:10px; border:1px solid #ddd; border-radius:5px; resize:vertical; min-height:80px; box-sizing:border-box;"></textarea>
           <button id="submitEventSurvey" style="width:100%; margin-top:10px; padding:10px; background:#4a90e2; color:#fff; border:none; border-radius:5px; cursor:pointer; font-weight:bold;">Submit Feedback</button>
         </div>
       `;
       
       // Add to page
       document.body.appendChild(surveyContainer);
       
       // Close button functionality
       document.getElementById('closeEventSurvey').addEventListener('click', function() {
         document.body.removeChild(surveyContainer);
       });
       
       // Rating button functionality
       var ratingButtons = document.querySelectorAll('.rating-btn');
       var selectedRating = null;
       
       ratingButtons.forEach(function(button) {
         button.addEventListener('click', function() {
           // Get the rating value
           selectedRating = this.getAttribute('data-rating');
           
           // Reset all buttons
           ratingButtons.forEach(function(btn) {
             btn.style.background = '#f9f9f9';
             btn.style.color = '#333';
             btn.style.borderColor = '#ddd';
           });
           
           // Highlight selected button
           this.style.background = '#4a90e2';
           this.style.color = '#fff';
           this.style.borderColor = '#4a90e2';
           
           // Show feedback area for ratings <= 3
           if (parseInt(selectedRating) <= 3) {
             document.getElementById('ratingFeedback').style.display = 'block';
           } else {
             // For positive ratings (4-5), submit immediately
             dataLayer.push({
               'event': 'customEventSurveyResponse',
               'eventType': eventType,
               'eventName': eventName,
               'surveyRating': selectedRating,
               'surveyFeedback': ''
             });
             
             // Thank user
             surveyContainer.innerHTML = '<div style="text-align:center; padding:20px;"><h3>Thank You!</h3><p>We appreciate your feedback!</p></div>';
             
             // Remove after delay
             setTimeout(function() {
               if (document.body.contains(surveyContainer)) {
                 document.body.removeChild(surveyContainer);
               }
             }, 3000);
           }
         });
       });
       
       // Submit button functionality (for negative ratings)
       var submitButton = document.getElementById('submitEventSurvey');
       if (submitButton) {
         submitButton.addEventListener('click', function() {
           var feedbackText = document.getElementById('eventFeedback').value || '';
           
           // Send the data
           dataLayer.push({
             'event': 'customEventSurveyResponse',
             'eventType': eventType,
             'eventName': eventName,
             'surveyRating': selectedRating,
             'surveyFeedback': feedbackText
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
       }
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Custom Event Survey Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `customEventSurveyTrigger`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your survey tag, select **Custom Event Survey Trigger**.
9. Click **Save**.

---

## Step 6: Create Data Collection Tag

1. Go to the **Tags** section.
2. Click the **New** button.
3. Name the tag: **Custom Event Survey Response Collector**.
4. Click **Tag Configuration**.
5. Choose **Custom HTML** tag.
6. Paste the following code (replace with your actual survey endpoint):

   ```html
   <script>
     (function() {
       // Get response data from dataLayer
       var dlEvent = {{dataLayer}};
       
       // Only proceed if this is a custom event survey response
       if (dlEvent.event !== 'customEventSurveyResponse') return;
       
       // Prepare data for sending
       var responseData = {
         type: 'customEventSurvey',
         eventType: dlEvent.eventType,
         eventName: dlEvent.eventName,
         rating: dlEvent.surveyRating,
         feedback: dlEvent.surveyFeedback,
         url: window.location.href,
         timestamp: new Date().toISOString(),
         userId: {{User ID}} || 'anonymous'
       };
       
       // Send to your survey endpoint
       fetch('https://your-survey-endpoint.com/api/custom-event-survey', {
         method: 'POST',
         headers: {
           'Content-Type': 'application/json',
         },
         body: JSON.stringify(responseData)
       })
       .then(response => response.json())
       .then(data => console.log('Custom event survey submitted:', data))
       .catch(error => console.error('Error submitting custom event survey:', error));
     })();
   </script>
   ```

7. Create a new trigger:
   - Navigate to **Triggers**.
   - Click **New**.
   - Name it **Custom Event Survey Response Trigger**.
   - Select **Custom Event**.
   - For **Event Name**, enter: `customEventSurveyResponse`.
   - Set it to fire on **All Custom Events**.
   - Click **Save**.

8. Under **Triggering** for your data collection tag, select **Custom Event Survey Response Trigger**.
9. Click **Save**.

---

## Step 7: Variable Setup for DataLayer Values

1. Go to **Variables** in your GTM workspace.
2. Click the **New** button under "User-Defined Variables".
3. Create the following variables:

   a. **dlv - eventType**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `eventType`

   b. **dlv - eventName**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `eventName`

   c. **dlv - formId**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `formId`

   d. **dlv - formName**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `formName`

   e. **dlv - videoId**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `videoId`

   f. **dlv - videoName**
   - Variable Configuration: Data Layer Variable
   - Data Layer Variable Name: `videoName`

---

## Step 8: Test and Publish

1. Click the **Preview** button in GTM.
2. Visit a page with the custom events you're tracking.
3. Trigger the custom event (submit a form, play a video, etc.).
4. Verify that the survey appears after the configured delay.
5. Test submitting different ratings and feedback.
6. Check that data is being collected properly.
7. If everything functions correctly, publish your changes.

---

## Examples of Custom Event Implementations

### Video Player Events

```html
<script>
  (function() {
    var videos = document.querySelectorAll('video, iframe[src*="youtube"], iframe[src*="vimeo"]');
    
    videos.forEach(function(video) {
      // For HTML5 videos
      if (video.tagName.toLowerCase() === 'video') {
        // Generate an ID if none exists
        if (!video.id) video.id = 'video-' + Math.floor(Math.random() * 1000);
        
        // Video name from data attribute or alt
        var videoName = video.getAttribute('data-video-name') || video.getAttribute('alt') || video.id;
        
        // Play event
        video.addEventListener('play', function() {
          dataLayer.push({
            'event': 'videoPlay',
            'videoId': this.id,
            'videoName': videoName
          });
        });
        
        // Complete event
        video.addEventListener('ended', function() {
          dataLayer.push({
            'event': 'videoComplete',
            'videoId': this.id,
            'videoName': videoName
          });
        });
      }
      
      // For YouTube iframes
      if (video.src && video.src.includes('youtube')) {
        // Implementation for YouTube API events
        // (Requires YouTube iframe API)
      }
    });
  })();
</script>
```

### Search Query Events

```html
<script>
  (function() {
    var searchForms = document.querySelectorAll('form[role="search"], form.search-form');
    
    searchForms.forEach(function(form) {
      form.addEventListener('submit', function(e) {
        // Find the search input
        var searchInput = this.querySelector('input[type="search"], input[name="s"], input[name="q"]');
        
        if (searchInput && searchInput.value) {
          dataLayer.push({
            'event': 'searchQuery',
            'searchTerm': searchInput.value,
            'searchLocation': window.location.pathname
          });
        }
      });
    });
  })();
</script>
```

---

## Optional Enhancements

- **Event-Specific Survey Questions:** Customize survey questions based on the specific event type and context.
- **Progressive Feedback:** For multi-step interactions, show different surveys at different stages of the process.
- **Conditional Survey Logic:** Only show surveys for certain events based on user segments or behavior patterns.
- **Session Limits:** Implement logic to limit the number of surveys a user sees in a single session.
- **Integration with User Journey:** Connect survey responses with the user's journey through your site for more context.
- **Dynamic Survey Timing:** Adjust when surveys appear based on user engagement signals.

--- 