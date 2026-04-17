

read and reference DESIGN.md.
https://www.portagecreekwebdesigns.com/ is the source of truth for design and aesthetic. 

# this is the Client Intake Portal for Portage Creek Web Designs. 

this is linked from the 'Sign In' button from the https://www.portagecreekwebdesigns.com/. 
on load, this will show the clerk sign in page. 

For user, 
- new user sign in using gmail/google SSO: the page takes to the intake quetions. intake questions have three steps. 1) business contact: business name, full name, phone number in 10 digits in this format (XXX)-XXX-XXXX. 2) tell me more about your business. what products /services are you prommoting, who is your ideal customer, what probelms are you solving, and what makes you different from your competitors. 3) design, build, & launch context: do you have brand designs and preferences. any reference sites you like or dislike? what's your budget?


- existing user, take them to the status page. Below is an example of the page. 


Project Status for { Full Name }
Track the progress of { Business Name }

1) Under Review 
We will reach out for more information, if needed.
2) Design & Scaffolding
Your website is being designed and built with custom code. 
3) Website is live
Preview your website for feedback. 
4) Payment Status 
Complete payment to proceed with the final launch. 
5) Launched!
Your website is live and ready for the world. 


For admin (portagecreekwebdesigns@gmail.com is the sole admin) 
- once log in, it takes to the dashboard page. 
- dashboard page has three parts. 1) client stats and 2) client records 3) pending sign up clients 
- client stats have four stats: 1) total clients,  2) Pending Sign Up, 3) In Progress, 4) Launched 
- admin can update 5 statuses above with notes attached to each status. 
- admin can delete client and related records.  


## tech stack 
- next.js 16 framework 
- typescript 
- neom + Drizzle ORM (ask me for .env keys)
- clerk for authentication (ask me for .env keys), clerk userId should be synced with neom. 
- suggest tech stack on how to send notifications to client and to admin for changes. 