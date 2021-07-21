
# Graphics
## sgplot
	Title1 "PABA Dissolution";
	Title2 "Identified by Drug Type";
	proc sgplot data = j.pvp2;
	VLINE DRatio /Response=PABA_Slope STAT = Mean
		LIMITS = BOTH LIMITSTAT=CLM ALPHA=0.10
		MARKERS group=PABA_Type;
	run;

## gplot
	goptions cback=white;
	axis1 minor=none label=( angle=90 'Median HR' );
	axis2 minor=none label=('Time Point');
	symbol interpol=HILOTJ c=red h=2;
	title 'Median HR : 6 to 12 min';
	proc gplot data=plotds;
		format Time1 TPointsSix.;
		plot  HR*Time1 /vaxis=axis1 haxis=axis2;
	run; quit;

## g3d
	proc g3d data=face002resultstrimmed;
		scatter RefPointX*_RefPointY=PNegative;
	run;

## sgscatter
	options orientation=landscape;
	ods pdf file="D:\pdx\subject45.pdf";
	proc sgscatter data=j.smoothed_both;
		where id = "45";
		by filename;
		plot (HRmed5 spdmed5)*(min) /
		markerattrs=(size=4) loess=();
	run;
	ods pdf close;

## sgpanel
	proc sgpanel data=j.smoothed_both;
		where ID = "45";
		panelby filename / columns=1;
		rowaxis grid;
		series x=min y=spdmed5;
	run;


# Data Manipulation

## import Excel file
	PROC IMPORT OUT= J.V3 
				DATAFILE= "J:\Statistics\Data 12-2.xlsx" 
				DBMS=EXCEL REPLACE;
		 RANGE="V3$"; 
		 GETNAMES=YES;
		 MIXED=NO;
		 SCANTEXT=YES;
		 USEDATE=YES;
		 SCANTIME=YES;
	RUN;

## merge
	DATA faceresults ; 
	  MERGE face.rightmeasure002 pointwiseresults_copy; 
	  BY ID; 
	RUN;

## sort
	proc sort data=compiled.smoothed_year1;
	by id exdate; run;

## retain first by group
	data compiled.baselinedates;
		set compiled.smoothed_year1;
		by ID;
		baselinedate = exdate;
		if first.id then output;
		keep ID baselinedate;
	run;

#  Summary Statistics
## freq
	proc freq data=j.corr3;
		where spearman < -0.8;
		table filename;
	run;

## means
	proc means data=j.smoothed_both_labeled_stopage noprint;
		where  6 <= min <= 12;
		by id  exdate;
		var hrmed5;
		output out=j.HRmeds6_12 median=HRSession612;
	run;

## univariate
	proc univariate data=compiled.smoothed_both;
		var resetsHR resetsSPD;
		histogram;
	run;

# Association / Regression
## correlation
	ods rtf file = "Correlations.rtf";
	proc corr data = wu.release spearman pearson;
		var DrugAmount
		PercentPolyWt
		PolymerAmount
		RateDrug
		RatePolymer
		RateRatio
		Ratio
		RelDrugRate
		RelPolyRate;
	run;
	ods rtf close;

## anova
	proc glm data=j.auc;
		title1 "ANOVA on P. ging";
		class treatment;
		model Time = treatment;
		means treatment / tukey cldiff;
		lsmeans treatment / adjust = tukey;
	run; quit;


## glm (regression)
	proc glm data = j.pvp2;
		class PABA_Type;
		model PABA_Slope = PABA_Type DRatio Dratio*PABA_Type / SOLUTION;
		lsmeans PABA_Type / ADJUST=TUKEY;
	run;

## logistic regression
	proc logistic data=j.logit descending;
		by treatment;
		model dead = time / covb ;
	run;


# Nonparametrics
## Wilcoxon Rank Sum Test (2 groups)
	proc npar1way wilcoxon;
		class caries;
		var HNP_1_95 HNP_2_95 HNP_3_95 HBD_2_95 HBD_3_95 LL_37_95;
		title3 "Comparison of viability by disease status";
	run;

## Kruskal Wallis (2+ groups, ANOVA on ranks) 
	proc npar1way wilcoxon;
		class main_type;
		var HNP_1_95 HNP_2_95 HNP_3_95 HBD_2_95 HBD_3_95 LL_37_95;
		title3 "Comparison of viability by group";
	run;



# Nonlinear Modeling
## nlin
### 4 Parameter Logistic Model
	*Fits the nonlinear model to each of the cytokines;
	*4 Parameter Logistic Model;
	proc nlin data=cyto.logstandards  MAXITER=1000;
		where cyto in ("IL-5", "IP-10", "MIP-1b");
		by plate cyto;
		parms Max = 4 Min = 0 EC50 = 2.2 Hill = 1;
		model logMFI = Max + (Min - Max) / (1 + (logConc / EC50)**Hill);
	run;



# Rater Reliability
## kappa
	proc freq data=twotrials;
		by rater;
		tables Q2*Z2 / agree;
		exact agree;
	run;


# SAS Macro
## simple example
	options mprint;
	%MACRO pulloutsubjects();
	%do i=1 %to 43; 
	%let id=%scan(&subject,&i);

		data compiled.subj&id;
		set compiled.smoothed_both (where=(id="&id"));
		run;

		proc sort data=compiled.subj&id;
		by id exdate block5sec;
		run;

	%end;
	%MEND pulloutsubjects;

	%let subject = 01;
	%pulloutsubjects();

# Miscellaneous 
## orthogonal polynomial coefficients
	proc iml;
		x={0 0.30103 0.47712125 0.60205999 0.69897 0.77815125};
		xp=orpol(x,5);
		print xp;
	run; quit;
