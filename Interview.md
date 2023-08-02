

Candidate has been Working at Amazon for ~3 years.
Good points:
* Has a good understanding of the domain in which she worked and was able to clearly explain said domain.
* Was able to answer all the follow-up questions regarding her previously worked projects
* Quickly understood the coding problem and proposed a rudimentary answer without much effort.
* Correctly coded the basic solution and explained the time and memory complexity accurately. Edge cases are also handled well.
* When given time to improve the solution, they were able to do so after some deliberation.

Bad points:
* Mediocre communication skills, especially their grasp on English language.


Onsite recommendation. For a G5 level position, candidate demonstrated good communication as well as problem solving and coding skills.
- Raises the bar at G5: None. 
- Meets the bar at G5: Communication, problem solving, Coding.
- Lowers the bar at G5: None.

Interview Transcript
Began with questions about current role and projects. Candidate has been working at Hulu for 10+ years and has a lot experience in IAM domain, including but not limited to user authentication, authorization, 3rd party OAuth2 integration, and 2FA. Currently candidate mainly works on 2FA projects at Hulu. When asked about the access token implementation in the OAuth2 project, candidate described the problem and solutions deep and clearly with explanation of the tradeoffs between different designs.

Went on to problem solving and coding - implement fake phone number check to support testing phone number login flow in production.

Candidate started by checking that the problem was well-understood - that we take a string consist of single phone numbers and phone number ranges, checking if a given phone number is in the list. Candidate first implemented a simple solution when input size is small: just store the static config string and check input number against each string in the config string. He then continued to scale his solution for larger input size. To optimize the performance, candidate chose to store the single numbers in a hash set while ranges in a sorted map sorted by range start. To check the input number, first check the single numbers map, then check the sorted map by checking the lower bound of the input number. 

Good points included:
- Candidate was able to explain his solution clearly before coding.
- Candidate started with simple solution before scaling his solution by a more performant solution
- Candidate wrote good test cases to verify her implementation. 

Bad points included:
- Candidate should know more about the limitation of her solution: input validation, edge cases, scalability. 
- Candidate sometimes needs help in analyzing the time complexity of different solutions. 

Final Recommendation
So in summary, candidate analyzed problems well and explained his solution clearly. Candidate has a lot of domain knowledge and experiences in IAM. Candidate should be a good addition to our team at this level.