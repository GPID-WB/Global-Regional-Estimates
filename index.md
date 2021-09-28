## [Global and Regional Estimates](index.md) --- [Regional Aggregates: AFE/AFW](afeafw.md) 

# Global and Regional Poverty Estimates

This code allows the user to replicate the global and regional poverty estimates published by the World Bank. This code uses the [PovcalNet command](https://worldbank.github.io/povcalnet/) to estimate global and regional poverty aggregates from country-level lined-up data. The methodological details behind this calculations and the country-level lined-up estimates can be found on the [PovcalNet website](http://iresearch.worldbank.org/PovcalNet/home.aspx). 

# Global and Regional Poverty Estimates
### Install necessary STATA commands
```stata
ssc install povcalnet
ssc install renvars
ssc install missings 
```
### Prepare auxiliary population data needed for calculations
```stata
//Regional total population: this includes population for all 218 countries including economies without a lined-up estimates for which the poverty headcount rate is assumed to be equal to the population-weighted regional average 

insheet using "http://iresearch.worldbank.org/PovcalNet/js/regionalpopulation.js", clear
split v1, p(, ]) g(a)
drop v1
drop if _n==1

replace a1="regioncode" if _n==1
drop if a1=="null"
renvars, map("population"+word(@[1], 1))
renvars populationregioncode, map(substr("@", 11, .))
drop population
drop if _n==1

reshape long population, i(regioncode) j(year)
destring population, replace
keep if year>1980 & year<2020
tempfile pop
save `pop'

//Regional identifiers as defined in PovcalNet 
povcalnet, clear year(last)
keep regioncode countrycode 
duplicates drop
tempfile regions
save `regions'

```
### Calculate Global and Regional Poverty Headcount Rates
```stata
povcalnet, fillgaps clear 	//load country-level lined-up estimates using povcalnet command
keep if coveragetype==3 | coveragetype==4 | countrycode=="ARG" | countrycode=="SUR"
keep countrycode year headcount population

//merge regioncodes
merge m:1 countrycode using `regions', nogen	//merge with regional identifiers

keep regioncode countrycode year headcount population  
 
collapse headcount [aw=population] , by(regioncode year) // calculate regional headcount rates
sort year regioncode

merge 1:1 regioncode year using `pop', nogen //merge with regional total population
gen poorpop=headcount*population

tempfile regional
save `regional'

bys year: egen globalpop=sum(population)

collapse globalpop (mean) headcount [aw=population], by(year) //calculate global headcount rates
ren globalpop population
gen poorpop=headcount*population
append using `regional'

replace regioncode="WLD" if regioncode==""

sort year regioncode
br if year==2017
tempfile aggregates

save `aggregates', replace 		//regional and global poverty headcounts
```
### Calculate Coverage 
Poverty rates for a region are only reported when the available surveys cover at least 50 percent of the population in that region (within a window of three years either side of the reference year). Global poverty estimates are reported only if data is representative of at least 50 percent of the population in low-income and lower-middle income countries (LIC/LMIC countries), where most of the poor live. This requirement is only applied to the global poverty estimate, not at the regional level. The World Bank classification of countries according to income groups in the line-up year is used.
```stata
//Population for all countries: including countries without a lined-up estimate for which the regional headcount is assumed
insheet using "http://iresearch.worldbank.org/PovcalNet/js/population.js", clear
split v1, p(, ]) g(a)
keep if a3!="" | a1=="SXM" | a1=="PSE"
drop v1
replace a1="countrycode" if _n==1
renvars, map("population"+word(@[1], 1))
renvars populationcountrycode, map(substr("@", 11, .))

drop population
drop if _n==1

reshape long population, i(countrycode) j(year)
destring population, replace
keep if year>1980 & year<2020
tempfile pop
save `pop'

//Regional - level total population: includes countries without a poverty estimate

insheet using "http://iresearch.worldbank.org/PovcalNet/js/population.js", clear
split v1, p(, ]) g(a)
keep a1 
drop if _n==1 | _n==2

gen region="EAP" if a1=="East Asia and Pacific"
replace region="ECA" if a1=="Europe and Central Asia"
replace region="LAC" if a1=="Latin America and the Caribbean"
replace region="MNA" if a1=="Middle East and North Africa"
replace region="OHI" if a1=="Other high Income"
replace region="SAS" if a1=="South Asia"
replace region="SSA" if a1=="Sub-Saharan Africa"

encode (region), gen(regioncode)
replace regioncode=regioncode[_n-1] if regioncode==.

keep if region==""
drop region
ren a1 countrycode
drop if countrycode=="null"

merge 1:m countrycode using `pop', nogen

decode regioncode, gen(regioncode1)
drop regioncode 
ren regioncode1 regioncode
 
tempfile all
save `all'

//Argentina Rural: Urban data used in global poverty, rural population gets regional headcount (see Methodology section in PovcalNet website)
povcalnet , fillgaps count(arg) clear
keep countrycode population year
ren population pop_urban
merge 1:1 countrycode year using `all'
drop if _merge==2
drop _merge
gen pop_rural=population-pop_urban
keep regioncode countrycode year pop_rural 
renam pop_rural population

append using `all'	
tab countrycode if year==1981

tempfile population
save `population' , replace

//income group classification data
import excel using "https://databank.worldbank.org/data/download/site-content/OGHIST.xls", clear sheet ("Country Analytical History") cellrange(A5:AI240) firstrow

rename A code
rename Banks economy
drop if missing(code)

// Creating income classifications for countries that didn't exist
// Giving Kosovo Serbia's income classification before it became a separate country
*br if inlist(code,"SRB","XKX")
foreach var of varlist FY08-FY09 {
replace `var' = "UM" if code=="XKX"
}
// Giving Serbia, Montenegro and Kosovo Yugoslavia's income classification before they become separate countries
*br if inlist(code,"YUG","SRB","MNE","XKX")
foreach var of varlist FY94-FY07 {
replace `var' = "LM" if inlist(code,"SRB","MNE","XKX")
}
drop if code=="YUG"
// Giving all Yugoslavian countries Yugoslavia's income classification before they became separate countries
*br if inlist(code,"YUGf","HRV","SVN","MKD","BIH","SRB","MNE","XKX")
foreach var of varlist FY89-FY93 {
replace `var' = "UM" if inlist(code,"HRV","SVN","MKD","BIH","SRB","MNE","XKX")
}
drop if code=="YUGf"
// Giving Czeck and Slovakia Czeckoslovakia's income classification before they became separate countries
*br if inlist(code,"CSK","CZE","SVK")
foreach var of varlist FY92-FY93 {
replace `var' = "UM" if inlist(code,"HRV","CZE","SVK")
}
drop if code=="CSK"
// Dropping three economies that are not among the WB's 218 economies
drop if inlist(code,"MYT","ANT","SUN")

// Changing variable names
local year = 1988
foreach var of varlist FY89-FY21 {
rename `var' y`year'
local year = `year' + 1
}
drop economy
// Reshaping to long format
reshape long y, i(code) j(year)
rename y incgroup_historical
replace incgroup_historical = "" if incgroup_historical==".."

// Assume income group carries backwards when missing
//only until 1978: 3 years window for first global number in 1981
expand 11 if year==1988
bysort code (year): replace year=1977+_n if year==1988
tab year

// Changing label/format
replace incgroup = "High income"         if incgroup=="H"
replace incgroup = "Upper middle income" if incgroup=="UM"
replace incgroup = "Lower middle income" if inlist(incgroup,"LM","LM*")
replace incgroup = "Low income"          if incgroup=="L"

label var incgroup_historical "Income group - historically"
label var code "Country code"
label var year "Year"

isid code year
ren code countrycode

save "CLASS.dta", replace


********************************************************************************
			//query povcalnet survey data for three years window
********************************************************************************

povcalnet, clear
//keep only national coverage
keep if coveragetype==3 | coveragetype==4 | countrycode=="ARG" | countrycode=="SUR"
//if both income and consumption, choose consumption
bys countrycode year: egen min=min(datatype)
keep if datatype==min
isid countrycode year
keep regioncode countrycode datayear
tempfile surveyyears
save `surveyyears' , replace

//calculate coverage: both regional and lic/lmic
forvalues y=1981/2019{
	
	
	//merge income classification
		
		use CLASS.dta, clear
		keep if year==`y'
		
		merge 1:m countrycode using `surveyyears' ,nogen
		//three years window
		replace year=`y' if year==.
		gen covered = (abs(year-datayear)<=3)
		
		//keep countries with a survey within three years window
		keep if covered==1
		keep countrycode covered
		duplicates drop
		tempfile covered 
		save `covered'

		use CLASS.dta, clear
		keep if year==`y'
		merge 1:m countrycode year using `population' 
		drop if _merge==2 //keep only reference year
		drop _merge
		bys countrycode year: egen min_pop=min(population)
		replace countrycode="ARG-Rural" if countrycode=="ARG" & min_pop<population //criterion to identify ARG-Rural
		drop min_pop
		
		merge 1:1 countrycode using `covered', nogen
		replace covered=0 if covered==.

		preserve
		collapse covered [aw=population], by(year)
		gen regioncode = "WLD"
		tempfile world
		save    `world'

		restore
		preserve
		// Income group
		gen LICLMIC = inlist(incgroup,"Low income","Lower middle income")
		keep if LICLMIC ==1
		collapse covered [aw=pop], by(year)
		gen incgroup = "LICLMIC"
		tempfile LICLMIC
		save    `LICLMIC'
		restore
		
		// Regional coverage
		collapse covered [aw=pop], by(year regioncode)
		append using `world'
		append using `LICLMIC'
		replace regioncode="WLD" if incgroup=="LICLMIC"
		tempfile year`y'
		save `year`y''
}

use `year1981', clear

forvalues y=1982/2019{
	append using `year`y''
}

//criteria for coverage
gen coverage=0
replace coverage=1 if covered>0.50


//both World and LICLMIC above 0.5 for coverage to be satisfied
collapse (min) coverage, by (regioncode year)	

// merge with aggregates calculated above
merge 1:1 regioncode year using `aggregates' ,nogen

replace headcount=. if coverage==0
replace poorpop=. if coverage==0
drop coverage
sort year regioncode

br 
```
