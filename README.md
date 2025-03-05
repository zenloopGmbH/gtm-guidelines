# Google Tag Manager Survey Implementation Guides

This repository contains step-by-step implementation guides for setting up various types of user feedback surveys using Google Tag Manager (GTM). Each guide provides detailed instructions, code snippets, and best practices for implementing different survey triggering mechanisms.

## Overview

Collecting user feedback at the right moment is crucial for understanding user experience and identifying areas for improvement. These guides help you implement surveys that trigger based on specific user behaviors, ensuring you gather feedback when it's most relevant.

## Guide Categories

The implementation guides are organized into the following categories:

### 1. Page-Specific Surveys
- **[Dedicated Page Feedback](1_page_specific_surveys/dedicated_page_feedback.md)**: Display surveys on specific pages after a user has spent time on them.
- **[Section-Based Feedback](1_page_specific_surveys/section_based_feedback.md)**: Trigger surveys when users engage with specific sections of your content.

### 2. Interaction-Triggered Surveys
- **[Button Click Surveys](2_interaction_triggered_surveys/button_clicks.md)**: Display surveys after users interact with specific buttons or CTAs.
- **[Custom Events Surveys](2_interaction_triggered_surveys/custom_events.md)**: Trigger surveys based on custom events like form submissions or video views.

### 3. Scroll or Time-Based Surveys
- **[Scroll Depth Surveys](3_scroll_time_based_surveys/scroll_depth.md)**: Show surveys when users have scrolled to a certain depth of the page.
- **[Time on Page Surveys](3_scroll_time_based_surveys/time_on_page.md)**: Display surveys after users have spent a specific amount of time on a page.

### 4. Inactivity or Interruption-Based Surveys
- **[Inactivity Trigger Surveys](4_inactivity_interruption_surveys/inactivity_trigger.md)**: Show surveys when users become inactive to identify potential confusion.
- **[Abandonment Feedback Surveys](4_inactivity_interruption_surveys/abandonment_feedback.md)**: Capture feedback when users abandon forms, carts, or other processes.

### 5. Post-Conversion Surveys
- **Order Confirmation Surveys**: Gather feedback after successful purchases.
- **Thank You Page Surveys**: Collect feedback after form submissions or other conversions.

## How to Use These Guides

Each implementation guide follows a similar structure:

1. **Overview & Requirements**: Purpose of the survey trigger and what you'll need.
2. **Step-by-Step Implementation**: Detailed instructions with code snippets.
3. **Testing & Validation**: Steps to verify correct implementation.
4. **Optional Enhancements**: Advanced customizations for more sophisticated use cases.

## Prerequisites

To implement these survey triggers, you'll need:

- A Google Tag Manager account with access to your website's container
- Basic understanding of HTML and JavaScript
- Access to publish changes to your GTM container
- A survey tool or form endpoint to collect the responses

## General Implementation Flow

While each guide has specific details, the general implementation process follows these steps:

1. **Set up triggers** in GTM to detect the appropriate user behavior
2. **Create custom tags** that inject the survey-related code
3. **Configure variables** to pass relevant data between tags
4. **Test thoroughly** in preview mode before publishing
5. **Monitor and refine** based on actual user interactions

## Best Practices

- Only show surveys to a percentage of your users to avoid survey fatigue
- Limit the frequency of surveys shown to the same user
- Keep surveys brief and focused on specific aspects of the experience
- Ensure surveys don't disrupt the primary user journey
- Test surveys across different devices and browsers
- Monitor survey display frequency and response rates

## Integration Options

While the guides provide the technical implementation for triggering surveys, they can be integrated with various survey tools:

- Your own custom survey forms
- Third-party survey platforms like SurveyMonkey, Typeform, or Google Forms
- Customer feedback platforms like Zenloop, Hotjar, or UserVoice
- Live chat solutions that can be triggered based on the same conditions

## Contributing

Feel free to suggest improvements to these guides by submitting pull requests or opening issues with your feedback.

## License

These implementation guides are provided under [MIT License](LICENSE). 