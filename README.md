<!--
Copyright 2023 Google LLC

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->
<img align="left" width="150" src="https://services.google.com/fh/files/misc/feedgen_logo.png" alt="feedgen_logo" /><br>

# FeedGen: Optimise Shopping feeds with Generative AI

[![GitHub last commit](https://img.shields.io/github/last-commit/google/feedgen)](https://github.com/google/feedgen/commits)
[![Code Style: Google](https://img.shields.io/badge/code%20style-google-blueviolet.svg)](https://github.com/google/gts)

**Disclaimer: This is not an official Google product.**

*FeedGen works best for up to 30k items. Looking to scale further? Check out the [Product Studio API](https://developers.google.com/product-studio/docs/onboarding) or consider [processing your feed in BigQuery](bigquery/README.md).*

[Overview](#overview) •
[Get started](#get-started) •
[What it solves](#challenges) •
[How it works](#solution-overview) •
[How to Contribute](#how-to-contribute) •
[Community Spotlight](#community-spotlight)

## Updates

* [July 2024]: Added guide to [feed optimisation using BigQuery](bigquery/README.md).
* [May 2024]: Added support for `gemini-1.5-pro` and `gemini-1.5-flash`
* [April 2024]
  * **IMPORTANT**: As of April 9 and as per the updated [Merchant Center product data specification](https://support.google.com/merchants/answer/14784710) please use `structured_title` and `structured_description` when importing FeedGen's output into Merchant Center instead of `title` and `description` respectively. Refer to [these instructions](#using-structured_title-and-structured_description) for details.
  * Added support for Gemini 1.5 pro (preview): `gemini-1.5-pro-preview-0409`. Please note that the model name may (breakingly) change in the future.
* [March 2024]
  * Renamed Gemini models to `gemini-1.0-pro` and `gemini-1.0-pro-vision`
  * Added support for retrieving JSON web pages
* [January 2024]: Added support for fetching product web page information and
  using it for higher quality title and description generation
* [December 2023]
  * Added support for Gemini models (`gemini-pro` and `gemini-pro-vision`)
  * Unified description generation and validation - now handled by a single
    prompt
  * Added support for [image understanding](#image-understanding) for higher
    quality title and description generation (only available with
    `gemini-pro-vision`)
  * Added LLM-generated titles which should avoid duplicate values at the
    possible loss of some attribute information
* [November 2023]: Added description validation as a separate component
* [October 2023]: Made title and description generation optional
* [August 2023]: Added support for [text-bison-32k](https://cloud.google.com/vertex-ai/docs/generative-ai/learn/models#32k_models)
* [June 2023]: Moved Colab variant to `v1` and switched to JS/TS on `main`

## Overview

**FeedGen** is an open-source tool that uses Google Cloud's state-of-the-art
Large Language Models (LLMs) to improve product titles, generate more
comprehensive descriptions, and fill missing attributes in product feeds. It
helps merchants and advertisers surface and fix quality issues in their feeds
using Generative AI in a simple and configurable way.

The tool relies on GCP's Vertex AI API to provide both zero-shot and few-shot
inference capabilities on GCP's foundational LLMs. With
[few-shot prompting](https://cloud.google.com/vertex-ai/docs/generative-ai/text/text-overview),
you use the best 3-10 samples from your own Shopping feed to customise the
model's responses towards your own data, thus achieving higher quality and more
consistent output. This can be optimised further by fine-tuning the
foundational models with your own proprietary data. Find out how to fine-tune
models with Vertex AI, along with the benefits of doing so, at this
[guide](https://cloud.google.com/vertex-ai/docs/generative-ai/models/tune-models).

> Note: Please check if your target feed language is one of the
[Vertex AI supported languages](https://cloud.google.com/vertex-ai/docs/generative-ai/learn/models#language_support)
before using FeedGen, and reach out to your Google Cloud or Account
representatives if not.

## Get Started

To get started with FeedGen:

1. Make a copy of the Google Sheets
[spreadsheet template](https://docs.google.com/spreadsheets/d/19eKTJrbZaUfipAvL5ZQmq_hoxEbLQIlDqURKFJA2OBU/edit#gid=1661242997)
1. Follow the instructions detailed in the `Getting Started` worksheet

## Challenges

Optimising Shopping feeds is a goal for every advertiser working with Google
Merchant Center (MC) in order to improve query matching, increase coverage, and
ultimately click-through rates (CTR). However, it is cumbersome to sift
through product disapprovals in MC or manually fix quality issues.

FeedGen tackles this using Generative AI - allowing users to surface and fix
quality issues, and fill attribute gaps in their feeds, in an automated fashion.

## Solution Overview

FeedGen is an Apps Script based application that runs as an HTML sidebar (see
[HtmlService](https://developers.google.com/apps-script/guides/html) for
details) in Google Sheets. The associated Google Sheets spreadsheet template is
where all the magic happens; it holds the input feed that needs optimisation,
along with specific configuration values that control how content is generated.
The spreadsheet is also used for both (optional) human validation and setting up
a **supplemental feed** in Google Merchant Center (MC).

> Generative Language in Vertex AI, and in general, is a nascent feature /
technology. We highly recommend manually reviewing and verifying the generated
titles and descriptions. FeedGen helps users expedite this process by providing
a [score](#scoring--evaluation) for both titles and descriptions (along with
detailed components) that represents how "good" the generated content is, along
with a Sheets-native way for bulk-approving generated content via data filters.

First, make a copy of the [template spreadsheet](#get-started) and follow the
instructions defined in the **Getting Started** section. The first step is to
authenticate yourself to the Apps Script environment via the **Initialise**
button as shown below.

<img src='./img/getting-started.png' alt='Getting Started' />

Afterwards, navigate to the **Config** worksheet to configure feed settings,
Vertex AI API settings (including an estimation of the
[costs](#vertex-ai-pricing-and-quotas) that will be incurred), and settings to
control the content generation.

<img src='./img/config.png' alt='Config' />

### Description Generation

Description generation works by taking the prompt prefix given in the **Config**
sheet, appending a row of data from **Input** and sending the result as a prompt
to the LLM. This gives you great flexibility in shaping the wording, style and
other requirements you might have. All data from **Input Feed** will be provided
as part of the prompt.

If a web page link is provided in the input feed, you may also check the
`Use Landing Page Information` checkbox to load and pass sanitised content of
the product's web page into the prompt. All `span` and `p` tags are extracted
from the fetched HTML content and concatenated together to form an additional
paragraph of information that is passed to the LLM in the prompt, along with
dedicated instructions on how to use this additional information. JSON web
responses will be used as-is without additional parsing. Furthermore, fetched
web page information is cached using Apps Script's
[CacheService](https://developers.google.com/apps-script/reference/cache) for a
period of 60 seconds in order to avoid refetching and reparsing the content for
the generation of titles (which is a separate call to the Vertex AI API).

*Optional*: You can also provide examples of descriptions in the **Few-shot**
examples section (see below). Those will be appended to the prompt prefix as
well and inform the model of how *good* descriptions look like.

The result is directly output as **Generated Description**

### Description Validation

Since LLMs have a tendency to hallucinate, there is an option to ask the model
(in follow-up instructions within the same prompt) if the generated description
meets your criteria. The model evaluates the description it just generated and
responds with a numerical score as well as reasoning. Example
*validation criteria and scoring* are provided to give some hints on how to
instruct the model to evaluate descriptions - e.g. it includes criteria as well
as example score values.

### Title Generation

Titles use few-shot prompting; a technique where one would select samples from
their own input feed as shown below to customise the model's responses towards
their data. To help with this process, FeedGen provides a utility Google Sheets
formula:

```sql
=FEEDGEN_CREATE_CONTEXT_JSON('Input Feed'!A2)
```

Which can be used to fill up the “Context” information field in the few-shot
prompt examples table by dragging it down, just as for other Sheets formulas.
This "Context" represents the entire row of data from the input feed for this
item, and it will be sent as part of the prompt to the Vertex AI API.

Afterwards, you must manually fill in the remaining columns of the few-shot
prompt examples table, which define the expected output by the LLM. These
examples are very important as they provide the basis upon which the LLM will
learn how it should generate content for the rest of the input feed. The best
examples to pick are products where:

* You can identify attributes in the existing title that match *column names* from your input feed.
* You can propose a new title that is better than the existing one (e.g. contains more attributes).
* The proposed title has a structure that can be reused for other products within your feed.

We would recommend adding at least one example per unique category within your
feed, especially if the ideal title composition would differ.

<img src='./img/few-shot.png' alt='Few-Shot' />

FeedGen defaults to using attributes from the input feed instead of generated
attribute values for composing the title, to avoid LLM hallucinations and ensure
consistency. For example, the value `Blue` from the input feed attribute
**Color** for a specific feed item will be used for its corresponding title
instead of, say, a generated value `Navy`. This behaviour can be overridden with
the `Prefer Generated Values` checkbox in the **Advanced Settings** section of
the *Title Prompt Settings*, and is useful whenever the input feed itself
contains erroneous or poor quality data.

Within this same section you can also specify a list of *safe* words that can be
output in generated titles even if they did not exist beforehand in your feed.
For example, you can add the word "Size" to this list if you would like to
prefix all values of the `Size` attribute with it (i.e. "Size M" instead of "M").

Finally, you can also specify whether you would like the LLM to generate titles
for you using the `Use LLM-generated Titles` checkbox. This allows the LLM
to inspect the generated attribute values and select which ones to concatenate
together - avoiding duplicates - instead of the default logic where *all*
attribute values will be stitched together. This feature should work better with
Gemini models than PaLM 2, as Gemini models have better reasoning capabilities
that allow them to better strick to prompt instructions over PaLM 2 models.
Furthermore, LLM-generated titles allow you to specify the desired length for
titles in the prompt (max 150 characters for Merchant Center), which was not
possible previously.

Like descriptions, you may also choose to load information from the provided web
page link and pass it to the LLM for the generation of higher quality titles.
This can be done via the `Use Landing Page Information` checkbox, and when
checked, all features extracted from the web page data will be listed under a
[new attribute](#feed-gaps-and-new-attributes) called **Website Features**. New
words that were not covered by existing attributes will then be added to the
generated title.

Now you are done with the configuration and ready to optimise your feed. Use the
top navigation menu to launch the FeedGen sidebar and start generating and
validating content in the **Generated Content Validation** worksheet.

<img src='./img/generated.png' alt='Generated' />

You would typically work within this view to understand, approve and/or
regenerate content for each feed item as follows:

* The **Generate** button controls the generation process, and will first
  regenerate all columns with an *empty* or *failed* status before continuing on
  with the rest of the feed. For example, clearing row 7 above and clicking
  *Generate* will start the generation process at row 7 first, before continuing
  on with the next feed item.
  * This means that you may clear the value of the **Status** column for any
    feed item in order to *regenerate* it.
  * To start from scratch, click **Clear Generated Data** first before
    *Generate*.
* If an error occurs, it will be reflected in the **Status** column as "Failed".
* Approval can be done in bulk via filtering the view and using the
  **Approve Filtered** button, or individually using the checkboxes in the
  **Approval** column. All entries with a score above 0 will already be
  pre-approved (read more about FeedGen's
  [Scoring / Evaluation](#scoring--evaluation) system below).
* Additional columns for titles and descriptions are grouped, so that you may
  expand the group you are interested in examining.

Once you have completed all the necessary approvals and are satisfied with the
output, click **Export to Output Feed** to transfer all approved feed items
to the **Output Feed** worksheet.

<img src='./img/output.png' alt='Output' />

The last step is to connect the spreadsheet to MC as a supplemental feed,
this can be done as described by this
[Help Center article](https://support.google.com/merchants/answer/7439058) for
standard MC accounts, and this
[Help Center article](https://support.google.com/merchants/answer/9651854)
for multi-client accounts (MCA).

Notice that there is a **att-p-feedgen** column in the output feed.
This column name is completely flexible and can be changed directly in the
output worksheet. It adds a custom attribute to the supplemental feed for
reporting and performance measurement purposes.

### Image Understanding

As Gemini (`gemini-pro-vision`) is a multimodal model, we are able to
additionally examine product images and use them to generate higher quality
titles and descriptions. This is done by adding additional instructions to the
existing title and description generation prompts for *extracting* visible
product features from the provided image.

For titles, extracted features are used in 2 ways:

1. As a quality check for existing feed attributes. For example, if the given
product feed references a *white* color, yet the provided image shows a *black*
product, the title and feed attribute are going to be adjusted accordingly.
2. To enhance the generated title via a
[new attribute](#feed-gaps-and-new-attributes) called **Image Features**. This
attribute lists all visible features the model was able to extract from the
provided image. All new words that were not covered by existing attributes will
then be added to the generated title.

For descriptions, extracted features are used by the model to generate a more
comprehensive description that highlights the visual aspects of the product.
This is particularly relevant for domains where visual appeal is paramount;
where the product's key details are *visually* conveyed rather than in a
structured text format within the feed. This includes fashion, home decor and
furniture, and perfumery and jewelry to name a few.

Finally, it is important to note the following restrictions (this information
is valid during the *Public Preview* of Gemini):

* You can specify either web images and/or Google Cloud Storage (GCS) file URIs
  in the `Image Link` column of the *Input Feed* worksheet. GCS URIs are passed
  as-is to Gemini (as they are supported by the model itself), while web images
  are first downloaded and provided inline as part of the input to the model.
* Regardless of the source, only `image/png` and `image/jpeg` MIME types are
  supported.
* GCS URIs must also point to a bucket that is within the same Google Cloud
  project that is sending the request (otherwise, it will be discarded by
  Gemini).
* [Pricing](#vertex-ai-pricing-and-quotas) is affected as well - an additional
  charge will be incurred per image. This has already been taken into account in
  FeedGen's price estimator.

### Scoring / Evaluation

#### Descriptions

Descriptions with a score below `Min. Evaluation Approval Score` will not be
pre-approved. You can re-generate those by filtering on **Description Score**
and removing the *Status* value in the **Generation Validation** tab.

#### Titles

FeedGen provides a score for generated titles between -1 and 1 that acts as a
quality indicator. Positive scores indicate varying degrees of good quality,
while negative scores represent uncertainty over the generated content. Like
descriptions, you may specify a minimum score (defaults to 0) that you would
like FeedGen to pre-approve.

Let's take a closer look with some fictitous examples to better understand the
scoring for titles:

* **Original Title**: 2XU Men's Swim Compression Long Sleeve Top
* **Original Description**: 2XU Men's Swim Compression Long Sleeve Top,
  lightweight, breathable PWX fabric, UPF 50+ protects you from the sun.
* **Generated Title**: 2XU Men's Swim Compression Long Sleeve Top Black Size M
  PWX Fabric UV Protection
* **Score**: -1
* Reasoning: New words, namely "UV Protection", were added that may have been
  *hallucinated* by the language model. Indeed, "UV Protection" is not
  explicitly mentioned anywhere in the input feed. Examining the feed item more
  clearly however surfaces that the description contains the value *UPF 50+*, so
  the addition of *UV Protection* is actually a *positive* thing, but since we
  have no way of assessing this (without applying a more granular semantic text
  analysis) we default to penalising the score.

Let's look at another example for the same product:

* **Original Title**: 2XU Men's Swim Compression Long Sleeve Top
* **Generated Title**: 2XU Men's Swim Compression Top Size M
* **Score**: -0.5
* Reasoning: Key information was removed from the title, namely "Long Sleeve",
  which describes the type of the product. FeedGen identifies this information
  by first examining the structure of titles via what we refer to as
  **templates** in our uniquitous language, before diving deeper and comparing
  the individual words that compose the titles. Let's check the templates for
  our example:
  * **Original Title Template**: `<Brand> <Gender> <Category> <Product Type>`
  * **Generated Title Template**: `<Brand> <Gender> <Category> <Product Type> <Size>`
  * As you can see no attributes were actually removed, but rather the
    components of the `Product Type` attribute changed in a worse way, hence the
    negative score.

> FeedGen is conservative in its scoring; it will assign a score of -0.5
  whenever *any* words get removed, even if those words were promotional phrases
  such as `get yours now` or `while stocks last`, which should not be part of
  titles as per the [Best Practices](#best-practices) outlined by Merchant
  Center (MC).

Alright, so what makes a good title? Let's look at another example:

* **Original Title**: 2XU Men's Swim Compression Long Sleeve Top
* **Generated Title**: 2XU Men's Swim Compression Long Sleeve Top Size M
* **Score**: 0.5
* Reasoning: Nothing was changed or lost in the original title, and we added a
  new attribute, "Size". If the product was being offered in different sizes,
  this new title would now be vital in preventing all feed items for the product
  from getting rejected by MC (due to their titles being duplicates).

Finally, what's the ideal case? Let's take a look at one last example:

* **Original Title**: 2XU Men's Swim Compression Long Sleeve Top
* **Input - Color**: *missing*
* **Generated Title**: 2XU Men's Swim Compression Long Sleeve Top Black Size M
* **Output - Color**: Black
* **Score**: 1
* Reasoning: This is the best possible scenario; we optimised the title and
  filled feed attribute gaps along the way, a score of 1 is definitely
  well-deserved.

So in summary, the scoring systems works as follows:

|Are there hallucinations?|Have we removed any words?|No change at all?|Have we optimised the title?|Did we fill in missing gaps or extract [new attributes](#feed-gaps-and-new-attributes)?|
|-|-|-|-|-|
|-1|-0.5|0|Add 0.5|Add 0.5|

FeedGen also applies some basic MC compliance checks, such as titles and
descriptions must not be longer than 150 and 5000 characters, respectively. If
the generated content fails these checks, the value
`Failed compliance checks` will be output in the **Status** column. As
mentioned above, FeedGen will attempt to regenerate `Failed` items first
whenever the **Generate** button is clicked.

### Feed Gaps and New Attributes

FeedGen does not just fill gaps in your feed, but might also create completely
**new** attributes that were not provided in the *Input Feed*. This is
controllable via the few-shot prompting examples in the *Config* sheet; by
providing "new" attributes that do not exist in the input within those examples,
FeedGen will attempt to *extract* values for those new attributes from other
values in the input feed. Let's take a look at an example:

|Original Title|Product Attributes in Original Title|Product Attributes in Generated Title|Generated Attribute Values|
|---|---|---|---|
|ASICS Women's Performance Running Capri Tight|Brand, Gender, Product Type| Brand, Gender, Product Type, **Fit**| ASICS, Women's Performance, Running Capri, Tight|

Notice here how the **Fit** attribute was extracted out of *Product Type*.
FeedGen would now attempt to do the same for all other products in the feed,
so for example it will extract the value `Relaxed` as *Fit* from the title
`Agave Men's Jeans Waterman Relaxed`. If you do not want those attributes to be
created, make sure you only use attributes that exist in the input feed for your
few-shot prompting examples. Furthermore, those completely new feed attributes
will be prefixed with **feedgen-** in the **Output Feed** (e.g. feedgen-Fit) and
will be sorted to the end of the sheet to make it easier for you to locate and
delete should you not want to use them.

### Best Practices

We recommend the following patterns for titles according to your business domain:

|Domain|Recommended title structure|Example|
|---|---|---|
|Apparel|Brand + Gender + Product Type + Attributes (Color, Size, Material)|Ann Taylor Women’s Sweater, Black (Size 6)|
|Consumable|Brand + Product Type + Attributes (Weight, Count)|TwinLab Mega CoQ10, 50 mg, 60 caps|
|Hard Goods|Brand + Product + Attributes (Size, Weight, Quantity)|Frontgate Wicker Patio Chair Set, Brown, 4-Piece|
|Electronics|Brand + Attribute + Product Type|Samsung 88” Smart LED TV with 4K 3D Curved Screen|
|Books|Title + Type + Format (Hardcover, eBook) + Author|1,000 Italian Recipe Cookbook, Hardcover by Michele Scicolone|

You can rely on these patterns to generate the few-shot prompting examples
defined in the FeedGen `Config` worksheet, which will accordingly steer the
values generated by the model.

We also suggest the following:

* Provide as many product attributes as possible for enriching **description** generation.
* Use **size**, **color**, and **gender / age group** for title generation, if available.
* Do **NOT** use model numbers, prices or promotional text in titles.

### Vertex AI Pricing and Quotas

Please refer to the Vertex AI
[Pricing](https://cloud.google.com/vertex-ai/pricing#generative_ai_models) and
[Quotas and Limits](https://cloud.google.com/vertex-ai/docs/quotas#request_quotas)
guides for more information.

### Using structured_title and structured_description

As of April 9, 2024 and as per the updated [Merchant Center product data specification](https://support.google.com/merchants/answer/14784710), users need to disclose whether generative AI was used to curate the text content for titles and descriptions.
The main challenge with this is that users cannot send both `title` and `structured_title`, or `description` and `structured_description` in the same feed, as the original column values will always trump the `structured_` variants.
Therefore, users need to perform an additional series of steps *after* exporting the approved generations into FeedGen's **Output Feed** tab:

1. Connect the FeedGen spreadsheet to Merchant Center as a [supplemental feed](https://support.google.com/merchants/answer/7439058).
1. Create [feed rules](https://support.google.com/merchants/answer/7450276) to **clear** the title and description values along with the FeedGen supplemental feed.
   1. You have to create 2 distinct feed rules as shown below:
   <img src='./img/feed_rule_title.png' width="300px" />
   <img src='./img/feed_rule_description.png' width="289.5px" />
1. Rename the `title` and `description` columns in the **Output Feed** tab of FeedGen to `structured_title` and `structured_description`, respectively.
1. Add the prefix `trained_algorithmic_media:` to all generated content.
   <br/><img src='./img/output_feed_adjustment.png' width="300px" /><br/>
   Refer to the detailed [structured_title](https://support.google.com/merchants/answer/6324415) and [structured_description](https://support.google.com/merchants/answer/6324468) attribute specs for more information.

We will be automating Steps #3 and #4 for you soon - stay tuned!<br/>
*Credits to [Glen Wilson](https://www.linkedin.com/in/glenmwilson/) and the team at [Solutions-8](https://sol8.com/) for the details and images.*

## How to Contribute

Beyond the information outlined in our [Contributing Guide](CONTRIBUTING.md),
you would need to follow these additional steps to build FeedGen locally:

1. Make sure your system has an up-to-date installation of
   [Node.js and npm](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm).
1. Navigate to the directory where the FeedGen source code lives.
1. Run `npm install`.
1. Run `npx @google/aside init` and click through the prompts.
   * Input the Apps Script `Script ID` associated with your target Google Sheets
     spreadsheet. You can find out this value by clicking on
     `Extensions > Apps Script` in the top navigation menu of your target sheet,
     then navigating to `Project Settings` (the gear icon) in the resulting
     [Apps Script](https://script.google.com) view.
1. Run `npm run deploy` to build, test and deploy (via
   [clasp](https://github.com/google/clasp)) all code to the target spreadsheet
   / Apps Script project.

## Community Spotlight

* [Unlocking the Power of AI for Google Shopping Feed Optimization](https://blog.datatovalue.nl/google-shopping-feed-optimization-with-generative-ai-16d0ed996f1f) by Krisztián Korpa.
* [AI-driven success: how to leverage the potential of Google FeedGen in PPC campaigns](https://www.adchieve.com/en/blog/leverage-potential-google-feed-gen/) by Alex van de Pol.
* (German) [Generative KI: Home24 verbessert mit FeedGen Reichweite und Performance von Shopping Ads](https://www.thinkwithgoogle.com/intl/de-de/marketing-strategien/automatisierung/home24-feedgen-shopping-ads) - Think with Google.
