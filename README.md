PatentLabeling
==============

This code was developed as part of the USPTO Algorithm Challenge, a competition organized by NASA and hosted by TopCoder.com with the purpose of digitalizing patents for the US Patent and Trademark Office.

In the Problem Statement section there is a brief description of the problem this code try to solve. In the Solution section the algorithm used to solve it is presented.

You can find more information on the competition at http://community.topcoder.com/ntl/ and the full problem statement at http://community.topcoder.com/longcontest/?module=ViewProblemStatement&rd=15027&pm=11645. Note that you should be a TopCoder member to be able to read the later.


Problem Statement
-----------------

For this problem a patent page composed by one or more figures together with a textual description are provided. Each figure in the patent is the image of a section of a patented system. Furthermore, each figure has exactly one title and a certain number of labels (possibly none).

You may refer to the [examples][examples] directory as well as to the link to the problem statement above for further information regarding the input.

The code must implement two methods: `getFigures()` and `getPartLabels()`. The former must return the best bounding box for each figure presents in a patent page together with its title, the later must return the best bounding box for each label on the patent page together with the its text.
  

Solution
--------

Since `getPartLabels()` is a sub-problem of `getFigures()`, for the purposes of this section is presented just the algorithm that implements the second method. You can always refer to the documented code for any detail.

The following pseudo-code explains in a nutshell how the code works:

	function result = getFigures(image, text)
	
	  imageBin = binarizeImage(image)
	  boxes = findBoundingBoxes(imageBin)
	  [patentOrientation titleBoxes labelBoxes otherBoxes] = scanImage(boxes)
	  matchTitlesWithHTML(patentOrientation, titleBoxes, text)
	  matchTitlesWithBoxes(patentOrientation, titleBoxes, labelBoxes, otherBoxes)

	end


### binarizeImage

Binarizing the image is necessary if we want to deal with grayscale image. For the purposes of this problem having 256 levels of gray does not bring more information than having 2 levels, while it may increase the complexity of the algorithm. Hence, the original image is scanned and each pixel on the image is replaced by 0 (black pixel) if its value is smaller than a fixed threshold (120), 255 (withe pixel) otherwise.


### findBoundingBoxes

The purpose of this method is to find a set of boxes on the image such that each box satisfies two rules. The first states that it must contain a set of all the black pixels in the image such that each pixel is recursively 8-connected with at least another in the set. Namely, all the pixels inside the final box at coordinates (x,y) must be the neighbor of at least another pixel at coordinates (x-1,y-1), (x,y-1), (x+1,y-1), (x+1,y), (x+1,y+1), (x,y+1), (x-1,y+1) or (x,y-1) and there cannot be pixels outside of the set that are neighbors of the pixels inside the set. The second rule state that it must be the smallest rectangular region that contains the set.

Afterward, given this set of boxes, it is possible to classify their content (letters or numbers) and consequently reassemble and naming each figure.


### scanImage

This is the main part of the algorithm. It is in charge of doing two jobs: find the orientation of the patent (portrait or landscape) and split the boxes in three sets w.r.t their content. In particular, it must classify each box as a figure's title, a figure's label or figure's section (neither a title nor a label).

For the sake of clarity this method is further decomposed in the following function calls:

	function [patentOrientation titleBoxes labelBoxes otherBoxes] = scanImage(boxes):

	  for orientation = [landscape portrait]
	    for box = all(boxes)
	      OCR(box)
	    end
	  end

	  charBoxes = mergeBoxes(boxes)
	  [titleBoxes labelBoxes] = findLabelHeight(charBoxes)
	  patentOrientation = computeOrientation(titleBoxes)

	end 

For both the possible patent orientation the content of all boxes is OCRed (read the sub-section below for further explanations). From now onward every boxes are associated with a certain probability of representing an alphanumeric char.
At this point is possible to merge boxes together in order to reconstruct a word. Indeed, mergeBoxes inspects boxes that are likely to be part of the same word and fusing them.

What it remains to be done is splitting the charBoxes set in a figures' title set and a figures' label set and find the orientation of the patent paper.
The former can be easily done noting that the title's font is always larger than the label's font. The later can be accomplished using the probabilities computed by the OCR. In particular, both orientations have a set of boxes - with cardinality >= 0 - containing titles. The probability of these boxes can be used for computing the paper orientation.


### OCR

The OCR is a 3-layer Artificial Neural Network (ANN) and each layer consists of: 400 nodes (20x20 pixels image) for the input layer, 105 nodes for the hidden layer and 70 nodes for the output layer.


### matchTitlesWithHTML

The purpose of this method is to correct misspelling in the figure's title. The textual description that is provided is a good help in doing it. 
In particular, the figures that are depicted inside a patent sheet are always in alphabetic order w.r.t the titles present in the description. For instance, if figures {"1" "2A" "2B" "3A" "3B" "3C" "4" "5"} are described in the paper, it is likely that the sequences {"2A" "2B" "3A"} or {"3A" "3B" "3C" "4"} are depicted, but not the sequence {"1" "3A" "4"}.
Then, the problem can be redefined as "Given an ordered sequence S of titles present in the HTML, find a sub-sequence T of S such that the probability of the titleBoxes matching T is maximized". Given that the likelihood that a box matches contains a title has already being computed this is an easy task.


### matchTitlesWithBoxes

At this point we have all we need to reassemble the figures. The approach this solution use is a modified k-nearest neighbor, where the initial cluster centers are the center of the title boxes. Instead of moving a cluster center w.r.t the positions of the members of a cluster, this algorithm take in consideration also the weight (area) of the members (boxes) and update the center of mass of the cluster center as a new box is merged.

