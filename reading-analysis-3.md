Reading Analysis #3
Paper: The Unexpected Dangers of Dynamic JavaScript

### What is the security problem the paper addresses? In what kind of threat model(s) does the problem exist?

The paper "The Unexpected Dangers of Dynamic JavaScript" is a defense paper that brings up how a majority of the top websites on the Internet are vulnerable to a type of attack called Cross Site Script Inclusion (CSSI), wherein the website customizes javascript with user information. Often this information contains sensitive data and session information, enabling attackers to hijack the information within them from another website as long as the user is logged into the original website (ie. the cookies were sent to the webserver and the javascript was customized with personal information). Any user that visits an attackers website is vulnerable if the original website generates javascript dynamically and includes personal or session information.

### How significant is the problem? Specifically, to what degree do existing solutions not work sufficiently well?

This CSSI is a large problem since the majority of websites evaluated were vulnerable to having sensitive information stolen from logged in users. The major reason for this being an issue is that the programmers of the website don't know that this vulnerability exists, therefore they continue to program websites with dynamic javascript.

### What is the defense? How does it work?

The authors of the paper introduce a way of defending from the CSSI by means of putting the sensitive user data in a separate file from the javascript code, thereby preventing the browser from interpreting it as javascript. The website then performs an AJAX call to get the file containing the sensitive user data. This separation allows the webserver to enforce Cross Origin Request Sharing (CORS), which only allows the same domain and allowed domains to access the file.

### To what degree will the defense potentially solve the targeted security problem? In particular, how difficult will it be for attackers to adapt to this defense?

The defense method introduced, as stated by the authors, will protect from the CSSI attack since attacker websites will be blocked from accessing the sensitive user data file because it's not in a javascript file and the file is protected by a restricted number of websites defined in the CORS headers. Attackers will have to figure out another way to get access to sensitive user data.

### What are the challenges facing deployment of the defense? Are they likely to be overcome?

The deployment of this defense method involves a proportional amount of work based on how the website developers designed the website or webapp. Everywhere in the code that relies on user information from an external javascript file will have to be updated to use the new method of having a non-javascript file. CORS policy might also have to be configured for other required websites to access the user data. The vulnerability could be introduced again if a developer were to add sensitive user data into an external javascript file.

### Did you like the paper?

I enjoyed reading the paper because I don't know much with web security beyond sanitizing input and not hardcoding password into html. AKA. I know enough javascript to be dangerous. "Access-Control-Allow-Origin: *" on my private API amiright?

### Was it easy to understand, or was it hard to read?

The paper was straightforward to understand, but it does require a basic knowledge of javascript and web browser security models. The paper does a good job of explaining what CORS and CSSI is. Unfortunately the diagrams figure 1 & 2 are hard to understand. I feel that a small HTML example would show the idea of CSSI better than what it currently is.

### Did you learn much from the paper?

I was surprised, I learned a lot more about javascript and its flaws. It's unfortunate that its so simple to write code that contains this CSSI security vulnerability. I also learned more about CORS protection, and how it's required to access content from a remote website via AJAX.

### How surprised were you by the result?

I was very surprised that a majority of the most popular websites on the Internet are vulnerable to having sensitive user information stolen, less the things that attackers can do to when they have that information (ie. change profile data, view private files, etc.). It's discouraging that only a few companies responded and fixed the CSSI issue on their website.
