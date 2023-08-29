# Explainer for Private Ad Measurement

## Introduction

Many websites rely on advertisements to fund their business and advertisers want to understand which of their advertisements and ad placements are most effective at driving business. Users want to browse the web without being tracked. Our goal is to define a system that can measure the effectiveness of advertisements on the Web without tracking users across websites.  

We try to build on previous privacy proposals such as [<u>Private Click Measurement</u>](https://webkit.org/blog/11529/introducing-private-click-measurement-pcm/) (PCM) , [<u>Interoperable Private Attribution</u>](https://github.com/patcg-individual-drafts/ipa/blob/main/IPA-End-to-End.md) (IPA), and [<u>Attribution Reporting API with Aggregatable Reports</u>](https://github.com/WICG/attribution-reporting-api/blob/main/AGGREGATE.md) (ARA). Our goal at each stage is to only transmit the minimum information necessary to perform the attribution measurement and nothing else.  

Like PCM, we rely on the user’s device to join an advertisement impression and conversion together. This means that the browser is trusted with the event level information on user interactions and joins them into summaries of attribution represented as *histograms*. These histograms only contain the attribution value of a conversion rather than a browsing history.  

Like IPA, we rely on Multi-Party Computation (MPC) frameworks to cryptographically segment data across multiple computation partners so that no individual organization can track an individual. This system is used for both aggregation and to introduce Differentially Private Noise and ensure that there is a well defined privacy loss bound for each user of the system. We rely on the Prio ([<u>Prio | Stanford Applied Crypto Group</u>](https://crypto.stanford.edu/prio/)) framework to perform this multi-party aggregation. We rely on Differential Privacy (([<u>Differential Privacy</u>](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/dwork.pdf), [<u>The Algorithmic Foundations of Differential Privacy</u>](https://www.cis.upenn.edu/~aaroth/Papers/privacybook.pdf))) to add appropriate noise to attribution calculations to make them private.  

Like ARA, we wish to allow measurements across a large, sparse space defining the potential linkages between advertisers and publishers. We present a concrete way to encode this sparse space using dense histograms so that individual contributions can be aggregated using known MPC approaches.  

## Example attribution

We want to support many different organizations measuring advertisements. Consider a system that keeps track of ad impressions, conversions, how the advertiser wants to attribute credit for a particular conversion, and what backend infrastructure will be used to process the information. In particular, the backend infrastructure includes *Prio* nodes that will perform multi-party aggregation and introduce differentially private noise to protect user privacy.  

### Ad Attribution

Advertisements are displayed to the user during the course of normal browsing. This is no different than it is today except that information about all of these ads is stored on the user’s device. The most familiar piece of this information is what would be needed to understand a user’s engagement with the ad, and how relevant it might be to any future purchase. Additionally, some metadata about the advertisement is stored so that it can be used for on-device attribution logic during a future conversion event.    

In order to include this particular ad in a conversion measurement, the device needs to know:  

- What advertisement system should be used for this ad?
- What advertiser will get the final attribution calculation?
- What is the identity (Ad ID) of this advertisement? Ads with the same Ad ID will be grouped together in the aggregation. This can be used to separate, for example, displayed ads from ad opportunities or compare different artwork on advertisements.
- How many times this ad might be used in an attribution calculation. More opportunities to use an ad give the ability to calculate user journeys (like how many dollars are spent in the first 7, 14, 21, and 28 days of opening an account) but require more noise to be added to each aggregate.
- How long should the device retain this ad for attribution?

This is retained in a database that is not accessible to any website. The database maintains a record of the advertisements a user has seen. Just as with a browsing history, this should be under the user’s control and able to be cleared.  

### Conversion

When a user performs a conversion, advertisement data is used to create attribution information for an advertiser. During the conversion, the advertiser communicates information to the browser that enables a calculation of attribution. The device needs to know:  

- What advertisement system should be used for this conversion?
- What advertiser is requesting the calculation? This should be the same advertiser who paid for ad placement
- What is the identity (Conversion ID) of this conversion? This will be used to group conversions together
- What is the value of this conversion?
- What advertisements (Ad IDs) should be considered in this conversion attribution calculation?
- What histogram bin should contain the attribution from each considered advertisement? Remember that eventually the reports will be histograms and this creates a way for every device to assign the same advertisement to the same bin in their individual histogram — and so each can be summed to find the aggregate attribution value.
- How many times might ads entering this calculation be used? Noise is added per Conversion ID and so any advertisement that is added must have a maximum number of uses less than or equal to the value specified by the conversion.
- What is the attribution logic to be used? Should this be last click? Last view? Some combination? This needs to be delivered to the device so that it can locally calculate the conversion.

Once this data is downloaded, the device can construct a histogram where each bin represents one or more advertisements and what fraction of the conversion value each should be attributed. In the case that there are no relevant advertisements, there should be a bin that stores value generated without advertising.    

### Reporting

Each histogram should be reported through a set of Prio nodes that are trusted by the browser. The Prio nodes have two responsibilities. First they aggregate all reports with the same Conversion ID. This means that there is no line level aggregation data on who contributed what to a particular aggregate. (See the original Prio description https://crypto.stanford.edu/prio/paper.pdf for more). Next, at least one of the nodes in the computation *that is not owned by the advertiser* must add noise to the histogram. The generation mechanism and scale of this noise should create appropriate Differential Privacy guarantees. This should be a published mechanism and scale so that signal to noise ratios can be calculated by each advertiser.  

The final result reported to an advertiser will be a histogram showing for each unique Conversion ID approximately what the summed attribution assigned to each relevant advertisement is. The sum is performed over all the users who underwent the particular conversion in a time window and the uncertainty is created by a fixed size noise added by the Prio system.

## Motivation

This system is designed to support advertisers with flexible queries while also protecting the users of a web browser and building transparency into advertising interactions.  

### Visibility and control of ad impression history

This system allows a user to view logs of any advertisement that they’ve encountered and, much like a browsing history, clear the logs before any report is sent off device.  

### Data collection and usage transparency

Because the join of impression and conversion events happens on device, the logic must be shared with the device. This gives the user a clear view of what data is collected and how it is used — and allows each user to audit the behavior of advertisers and ad-tech. This also adheres to a data minimization paradigm where only the data required at each step of the process is actually shared.  

### Building on internet standards

Prio type aggregations standards are being drafted by the [<u>IETF Privacy Preserving Measurement Working Group</u>]([https://datatracker.ietf.org/wg/ppm/](https://datatracker.ietf.org/wg/ppm/about/)). Using this type of underlying technology means that ad attribution will benefit from the ecosystem supporting a wide variety of measurements.
