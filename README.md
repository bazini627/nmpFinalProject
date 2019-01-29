# nmpFinalProject
Repo for NMP Masters Project

General idea for my final project is revamping [this](https://maps.calsurv.org/) web map.  I wrangle this data on a weekly basis, although the data I end up scraping doesn't have a geographic component like the point data in this 
map and is just in table form [here](http://www.westnile.ca.gov/web_reports.php?report=sle&option=print&year=2018), for some other folks who prepare a report for a regular meeting.  West Nile Virus is pretty well known at this point but we've seen a reemergence 
here in California of St. Louis Encephalitis since about 2015.  

I'm thinking about using this data in conjunction with some CDC data for census tracts that have been deemed vulnerable according to their [Social Vulnerability Index](https://svi.cdc.gov/).  With this map in it's current incarnation we can see where WNV/SLE 
outbreaks may be occurring and with my project I'd like to make an attempt at showing who may be affected the most by these viruses.  I have a vague idea of a dashboard which would basically just use a point-in-polygon analysis to display how many positive 
detections exist within a given SVI for a particular county. I'd also like to maybe show trends for WNV/SLE for CA and counties as another component of the dashboard.      
