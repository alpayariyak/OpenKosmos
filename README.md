# GrIT
A subset of GrIT can be downloaded at https://aka.ms/kosmos-2.

# From Kosmos-2 Paper:
## Construction of Web-Scale Grounded Image-Text Pairs (GrIT)

We introduce GrIT2, a large-scale dataset of **G**rounded **I**mage-**T**ext pairs, which is created based on image-text pairs from a subset of COYO-700M [1] and LAION-2B [19]). We construct a pipeline to extract and link text spans (_i.e._, noun phrases and referring expressions) in the caption to their corresponding image regions. The pipeline mainly consists of two steps: generating noun-chunk-bounding-box pairs and producing referring-expression-bounding-box pairs. We describe these steps in detail below:

### Step-1: 
Generating noun-chunk-bounding-box pairsGiven an image-text pair, we first extract noun chunks from the caption and associate them with image regions using a pretrained detector. As illustrated in Figure 3, we use spaCy [10] to parse the caption ("_a dog in a field of flowers_") and extract all noun chunks ("_a dog_", "_a field_" and "_flowers_"). We eliminate certain abstract noun phrases that are challenging to recognize in the image, such as "_time_", "_love_", and "_freedom_", to reduce potential noise. Subsequently, we input the image and noun chunks extracted from the caption into a pretrained grounding model (_e.g._, GLIP [19]) to obtain the associated bounding boxes. Non-maximum suppression algorithm is applied to remove bounding boxes that have a high overlap with others, even if they are not for the same noun chunk. We keep noun-chunk-bounding-box pairs with predicted confidence scores higher than 0.65. If no bounding boxes are retained, we discard the corresponding image-caption pair.

 ### Step-2: 
Producing referring-expression-bounding-box pairsIn order to endow the model with the ability to ground complex linguistic descriptions, we expand noun chunks to referring expressions. Specifically, we use spaCy to obtain dependency relations of the sentence. We then expand a noun chunk into a referring expression by recursively traversing its children in the dependency tree and concatenating children tokens with the noun chunk. We do not expand noun chunks with conjuncts. For noun chunks without children tokens, we keep them for the next process. In the example shown in Figure 3, the noun chunk '_a dog_' can be expanded to "_a dog in a field of flowers_", and the noun chunk '_a field_' can be expanded to "_a field of flowers_".

Furthermore, we only retain referring expressions or noun chunks that are not contained by others. As shown in Figure 3, we keep the referring expression "_a dog in a field of flowers_" and drop "_a field of flowers_" (as it is entailed by "_a dog in a field of flowers_") and '_flowers_'. We assign the bounding box of the noun chunk ('_a dog_') to the corresponding generated referring expression ("_a dog in a field of flowers_").

In the end, we obtain approximately 91M images, 115M text spans, and 137M associated bounding boxes. We compare GrIT with existing publicly accessible visual grounding datasets in Table 1. Data samples of GrIT are shown in the Appendix.

![data_generation_pipeline.png](images%2Fdata_generation_pipeline.png)

## Kosmos-2: A Grounded Multimodal Large Language Model

Kosmos-2 is a grounded multimodal large language model, which integrates grounding and referring capabilities compared with Kosmos-1. The model can accept image regions selected by the user using bounding boxes as input, provide visual answers (_i.e._, bounding boxes), and ground the text output to the visual world. Kosmos-2 adopts the same model architecture and training objective as Kosmos-1. We add grounded image-text pairs into the training data to endow the model with grounding and referring capabilities. For a text span (such as noun phrase and referring expression) and its corresponding bounding boxes in a grounded image-text pair, We discretize continuous coordinates of bounding boxes into a sequence of location tokens to encode with text tokens in a unified way. Then we link the location tokens and their corresponding text span via a "_hyperlink_" data

text pair, We discretize continuous coordinates of bounding boxes into a sequence of location tokens to encode with text tokens in a unified way. Then we link the location tokens and their corresponding text span via a "_hyperlink_" data format. The model is trained to establish a mapping between image regions and their corresponding location tokens and connect the image regions with their associated text spans.

| **Dataset**         | **Images**   | **Objects**  | **Text Spans** | **Avg Expression Length** |
|---------------------|--------------|--------------|----------------|---------------------------|
| Flickr Entities (PWC+15) | 31,783      | 275,775      | 513,644        | -                         |
| RefCOCOg (MHT+15)   | 26,711      | 54,822      | 85,474         | 8.43                      |
| RefCOCO (YPY+16)    | 19,994      | 50,000      | 142,209        | 3.61                      |
| RefCOCO+ (YPY+16)   | 19,992      | 49,856      | 141,564        | 3.53                      |
| Visual Genome (KZG+16)| 108,077     | 4,102,818   | -              | -                         |
| **GRIT**      | 90,614,680  | 137,349,210 | 114,978,233    | 4.7                       |


### Grounded Input Representations

Given a text span and its associated bounding boxes in a grounded image-text pair, we first convert the continuous coordinates of bounding boxes into a sequence of discrete location tokens [CSL+21]. For an image with width \(W\) and height \(H\), we evenly divide both the width and height into \(P\) segments each. \(P \times P\) bins are obtained and each bin consists of \(\frac{W}{P} \times \frac{H}{P}\) pixels. For each bin, we use a location token to represent the coordinates within that bin. We use the coordinates of the center pixel of each bin to determine bounding boxes on the image. In total, \(P \times P\) location tokens are introduced, and these tokens are added to word vocabulary to enable unified modeling with texts.

The bounding box can be represented using its top-left point (\(x_1\), \(y_1\)) and bottom-right point (\(x_2\), \(y_2\)). We discretize the top-left and bottom-right corner points to location tokens, respectively. We concatenate the top-left location token `<loc1>`, the bottom-right location token `<loc2>`, and special boundary tokens `<box>` and `</box>`, to represent a single bounding box: "`<box><loc1><loc2></box>`". If the text span is associated with multiple bounding boxes, we use a special token `<delim>` to concatenate the location tokens of these bounding boxes: "`<box><loc1><loc2><delim>...<loc1><loc2></box>`".

Then we arrange the text span and its associated location tokens in a format resembling a _"hyperlink"_ in markdown. For the text span with a single bounding box, the resulted sequence is "`<p>_text span_ </p><box><loc1><loc2></box>`", where `<p>` and `</p>` are special tokens indicating the beginning and end of the text span. The data format tells the model that image regions within the bounding box are associated with the text span.

