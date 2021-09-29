# Regional Poverty Estimates for East and West Africa regions
This code allows the user to replicate the poverty estimates for the East and West Africa regions as defined by the World Bank and reported in ["March 2021 PovcalNet Update- What's New"](https://documents1.worldbank.org/curated/en/654971615585402030/pdf/March-2021-PovcalNet-Update-What-s-New.pdf). The auxiliary material needed to run this code can be found [here](https://github.com/PovcalNet-Team/AFEAFW)

## Install necessary STATA commands

```stata
local cmds "renvars povcalnet _gwtmean"

foreach cmd of local cmds {
	cap which `cmd'
	if (_rc) ssc install `cmd'
}
```
## Prepare auxiliary data needed for calculations
```stata

//population data 
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
keep if year>1980
tempfile pop
save `pop'


//1.2 Get regional identifiers from povcalnet: this is needed because when querying lined-up estimates regional identifiers are not displayed
insheet using "http://iresearch.worldbank.org/PovcalNet/js/population.js", clear
split v1, p(, ]) g(a)
keep a1 
drop if _n==1 | _n==2

gen 	region="EAP" if a1=="East Asia and Pacific"
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
decode region, gen(regioncode1)
drop regioncode
ren regioncode1 regioncode


//AFE/AFW classification:
local afe_codes AGO BWA BDI COM COD ERI SWZ ETH KEN LSO    /// 
                MDG MWI MUS MOZ NAM RWA STP SYC SOM ZAF    /// 
                SSD SDN TZA UGA ZMB ZWE

local afe_codes = itrim("`afe_codes'")
local afe_codes: subinstr local afe_codes " " "|", all

local afw_codes BEN BFA CPV CMR CAF TCD COG CIV GNQ GAB /// 
                GMB GHA GIN GNB LBR MLI MRT NER NGA SEN SLE TGO
local afw_codes = itrim("`afw_codes'")
local afw_codes: subinstr local afw_codes " " "|", all

gen regioncode2 = ""
replace regioncode2="AFE" if regexm(countrycode, "`afe_codes'")
replace regioncode2="AFW" if regexm(countrycode, "`afw_codes'")

tempfile all
save `all'
```
## Calculate regional estimates
```stata
//povline: 1.90
foreach pl in 1.90 3.20 5.50{
	povcalnet, clear fillgaps region(ssa) povline(`pl')

	keep countrycode year headcount population 
	merge m:1 countrycode year using `all', nogen

	keep if regioncode=="SSA"

	//Countries missing data get the population-weighted regional average

	bys regioncode year: egen avg_ssa=wtmean(headcount), weight(population)
	replace headcount=avg_ssa if headcount==.

	gen poor=population*headcount
	bys regioncode2 year: egen poorpop=sum(poor)

	collapse poorpop (mean) headcount [aw=pop] , by (year regioncode2)
	replace headcount=headcount*100

	//apply coverage rule: only report a number if more than 50% pop in +/- 3years window of ref year
	merge 1:1 regioncode2 year using coverage, nogen 
	replace headcount=. if coverage<50
	replace poorpop=. if coverage<50
	drop coverage
	reshape wide poorpop headcount, i(year) j(regioncode2,string)

	label var poorpopAFE "East Africa- poor pop US$`pl'"
	label var poorpopAFW "West Africa- poor pop US$`pl'"
	label var headcountAFE "East Africa- headcount US$`pl'"
	label var headcountAFW "West Africa- headcount US$`pl'"
	export excel using "AFEAFW.xlsx", sheet("US$`pl'", modify) firstrow(varl)
}

exit 
```
