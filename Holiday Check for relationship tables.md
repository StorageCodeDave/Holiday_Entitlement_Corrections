# Holiday_Entitlement_Corrections
Shows Totals using existing data in tables and checks if the stored balance is correct.

It would compare other tables to see if the person existed and would highlight as deleted User. Having data on staff that had been deleted elsewhere had no impact on calculations, but wanted to have this table 100% accurate. I was not allowed to add any extra functions to the database so there are parts of the script that can be seen as repetitive. I new function was built during my use of this script called dbo.fnCurrentEntitlement in Picture3 called RTamount which I added to the script to compare to my calculations. I still found it was not fully correct due to how many factors that had to be considered to calculate.
P1
![image](https://github.com/user-attachments/assets/4a96c6f0-4136-4262-9233-7685218fbbab)


It has 2 error columns and if corrected 1 issue and there is still more it would change to the next error using a case statement. Here you can see the values stored in the table and my error messages. Note my error messages also include the correct value calculated off picture which are ready to transfer. Attention to detail can be seen when looking at this. There is data stored in it for 2023 for Brian and this is 2025 data. Benny has 20 days, booked 9 and remaining is 16 which is wrong. Cillian start date is 15th Jan so any holiday/Bank Holidays will not be counted, and last example is Edurado has 2 holiday for 26/07/24 found in Absences table and will count as 2 holidays
P2
![image](https://github.com/user-attachments/assets/e6c299b1-b8a5-405c-a317-c7fe2a4e3aaa)
 

Here are my calculations. Note to calculate the remaining balance you must calculate many factors as follows. Pro rota staff that start mid-year, calculate in Hours or days, did they work a bank holiday and auto add a holiday is switched on in this database,  is the starting amount the holiday group amount or have the User selected to manually change the start amount themselves, is it a ½ day or full day or hours you are subtracting and do they receive Long service award and extra holidays on length of service. 

P3
 
![image](https://github.com/user-attachments/assets/7b354b15-91dd-491a-9a58-eff79d4bd9bc)


IF object_ID ('tempdb..#entitlementstemp') IS NOT NULL
  DROP TABLE #entitlementstemp
go
/*
Entitlements

Error 1 & 2
Empref not in Realtime = Employee likely deleted but left in this table.
Deleted User = Employee likely deleted but left in this table.
DB Taken = The taken value is different but may look ok in Realtime reports
Type is wrong = The holiday Hrs or Days is wrong and may cause booking holiday issue.
Ent Amount wrong = Holiday Total is wrong, check if Control By column is set to User which mean Use Default is Unticked.
DB Booked = Holidays booked is wrong and will affect Realtime and Portal
Ent amount is NULL = Holiday total has no value. Use Check column to see if using Contrual holidays as stored else where.
Abs Booked is NULL = Holiday Booked is NULL, likely as not booked any Holiday but NULL value may cause reports to fail
Balance is NULL = The balance is self generated in reports and may populate once holiday start been booked 
DB bal = The balance is wrong but self generated in reports, but will cause major issue if wrong during transfer to the New Year script runs.
Manually changed amount = Use default is unticked and User changed the Holiday total.
Statutory XX & Contractual XX = User is using both Contractual and Statuary holidays and likely in Error.
ProRota StartDate = The Holiday amount stored is not the true starting value.
*/
declare @additionalBKHent int = (select AdditionalBKHEnt from system),@countDates int

	select coalesce(EMP.forenames,cast(EH.empref as Varchar(10))+' Empref not in')as forenames,coalesce(EMP.surname,'Empdetails')surname,EH.empref,EMP.Payrollno,
	case when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate and coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate) >=EH.entenddate then 'Full Year' else
	CONVERT(VARCHAR(8), p.startdate, 3) END AS [Start Date],
	Case when coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),'1899-12-30 00:00:00.000') = '1899-12-30 00:00:00.000' then CONVERT(VARCHAR(8),EH.entenddate,3) else '<'+CONVERT(VARCHAR(8),P.enddate,3)+'>' End As [Holiday End],
	--CONVERT(VARCHAR(10), (coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),'<'+CONVERT(VARCHAR(8),EH.entenddate,3)+'>')), 3) AS [Holiday End],
	case when Eh.defaultent = 1 then 'Realtime' else 'User' end as [Control By],
	Case when EH.HolGroupRef = -1 then 'Default' else AG.groupname+'  --Group '+cast(EH.HolGroupRef as Char(4)) end as		[Holiday Name],
	--EH.entamount as [DB entamount],
	--coalesce(ED.ContractedHoliday,'') as ContractedHoliday,
	cast(EH.entamount as Varchar(10)) + case when coalesce(ED.ContractedHoliday,0) = 0 then '' else '  Con =' + cast(coalesce(ED.ContractedHoliday,'') as varchar(10)) END as EntAmounts,
	Eh.broughtfwd as [DB Brt fwd1],
	EH.LSAEntAmount as [DB LSA],
	EH.additionalent as [DB Worked BH],
	coalesce(EH.broughtfwd,0)+coalesce(EH.entamount,0)+coalesce(EH.LSAEntAmount,0) as [DB B/F +Ent +LSA],
	'-' as [-],
	EH.absbooked,
	eh.calabstaken as DBTaken,
	Case when EH.enttype = 1 then 'Hour =' else 'Days =' End as [Type1],
	EH.balance as DBBalance,
/*use these to view why they are flagging but look the same. Slight rounding of 0.5 can happen and trigger
Visuals to highlight if different
*/
	Case 
		when EH.empref = Unpaid.empref then Unpaid.[Shift Name] +' shift pays Unpaid for Holiday'
		when DupGroup.Entref2 >0 then cast(DupGroup.Entref2 as Varchar (6))+ ' Entref Groupref = Holgroupref of Holiday'
		When EMP.payrollno IS NULL then 'Deleted User'
		when Eh.enttype <> ED.enttype then 'Type is wrong'
		when EH.entamount <> coalesce(ED.entamount,0) and eh.defaultent = 0 then 'Manually changed amount '+Cast(ED.entamount as Varchar(10))+' to '+Cast(EH.entamount as Varchar(10)) 
		when EH.entamount <> coalesce(ED.entamount,0) then 'Ent Amount wrong'
		when EMP.HolEntGroupRef <> coalesce(NullIF(EH.HolGroupRef,-1),1) then 'Empdetails Hols is different'
		when EH.empref = Ternimated.empref then 'Returned after leaving on '+CONVERT(VARCHAR(8), Ternimated.endDate, 3)
		when ED.ContractedHoliday > 0 and ED.entamount > 0 then 'Statutory '+Cast(EH.entamount as Varchar(10))+' & Contractual '+Cast(ED.ContractedHoliday as Varchar(10))
		when coalesce(EH.abstaken ,0) >0 then 'Abstaken also has = ' + cast (EH.abstaken as char(10))
		--when Coalesce(p.startdate,EH.entstartdate) >=EH.entstartdate then 'ProRota StartDate = '+CONVERT(VARCHAR(8), Coalesce(p.startdate,EH.entstartdate), 3)
		when Coalesce(p.startdate,EH.entstartdate) >EH.entstartdate then 'ProRota = '+cast(CEILING(DATEDIFF(DAY, Coalesce(p.startdate,EH.entstartdate), EH.entenddate) * (coalesce(ED.entamount,0) + coalesce(ED.ContractedHoliday,0)) / 365)as Varchar(10))+
		' Date = '+CONVERT(VARCHAR(8), Coalesce(p.startdate,EH.entstartdate), 3)
		when EH.entstartdate <> ED.entstartdate then 'Start date '+CONVERT(VARCHAR(8), EH.entstartdate, 3)+' Default '+CONVERT(VARCHAR(8), ED.entstartdate, 3)
		when coalesce(EH.absbooked,0)<>coalesce(AbsBookedTotal.AbsTotal,0) then 'DB Booked ='+cast (coalesce(EH.absbooked,0) as char(10))+' '+'Calc ='+cast (coalesce(AbsBookedTotal.AbsTotal,0) as char(10))
		when TakenSum.TakenDays <> coalesce(EH.calabstaken,0) then 'DB Taken='+cast (coalesce(EH.calabstaken,0) as char(10))+' '+'Calc ='+cast (TakenSum.TakenDays as char(10))
		when coalesce(NullIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate) <EH.entenddate then 'Leaver ' +CONVERT(VARCHAR(8), P.enddate, 3)
		--when coalesce(AbsBookedTotal.AbsTotal,0)=cast (coalesce(EH.absbooked,0) as decimal(10,2)) then '' 
		when p.startdate IS NULL then 'Personnel Start date is Null'
		when coalesce(Eh.entamount,0) = coalesce(ED.entamount,0) + coalesce(ED.ContractedHoliday,0) and Eh.defaultent = 0 then 'Default given but unticked'
		When Eh.defaultent = 0 and (p.startdate >=EH.entstartdate) then 'No Pro Rota used, default unticked'
		
		when Emp.HolEntGroupRef = 0 and (P.enddate = '1899-12-30 00:00:00.000' or P.enddate IS NULL or P.enddate>getdate()) then 'Empdetails holiday = 0 and Active'
	else '' END--'DB Booked ='+cast (coalesce(EH.absbooked,0) as char(10))+' '+'Calc ='+cast (coalesce(AbsBookedTotal.AbsTotal,0) as char(10)) end 
	as [Warnings],

	Case
		when EH.empref in (DupCheck.empref) then 'Duplicate Holidays on ' +CONVERT(VARCHAR(8), DupCheck.absdate, 3)
		when EH.entamount IS NULL and ED.entamount >0 then 'Ent amount is NULL'
		when EH.absbooked IS NULL then 'Abs Booked is NULL'
		when EH.balance IS NULL then 'Balance is NULL'
		when EH.balance =
			round
				(case 
				when coalesce(NullIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entstartdate) < EH.entstartdate then 0
				when Coalesce(p.startdate,EH.entstartdate) >EH.entstartdate and dbo.fnIsLeaver (EH.empref) = 1-- Start late Finished Early
				then CEILING(DATEDIFF(DAY, Coalesce(p.startdate,EH.entstartdate), coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
				when Coalesce(p.startdate,EH.entstartdate) >EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 0-- Start late only and Active
				then CEILING(DATEDIFF(DAY,p.startdate, coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
				when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 1 -- Started and Now Inactive
				then CEILING(DATEDIFF(DAY,EH.entstartdate, coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
				when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 0 then 
					case when EH.defaultent = 0 -- give full amount if joined full year
						then EH.entamount 
							else coalesce(ED.entamount,0)
								END +coalesce(ED.ContractedHoliday,0) 
					end +coalesce(EH.broughtfwd,0)+[dbo].[fnEmployeeLSA](EH.empref,EH.sysyear,case when EH.HolGroupRef = -1 then 1 else EH.HolGroupRef end)+ coalesce(BankHolsWorked.[N.W.Day get only worked hrs],BankHolsWorked.CountDate,0)
						-coalesce(AbsBookedTotal.AbsTotal,0) 
							,2)
		then '' 
		else 'DB bal = '+cast (EH.balance as varchar(10)) +' Calc = '+ 
			cast(
				round
				(	case 
		when coalesce(NullIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entstartdate) < EH.entstartdate then 0
		when Coalesce(p.startdate,EH.entstartdate) >EH.entstartdate and dbo.fnIsLeaver (EH.empref) = 1-- Start late Finished Early
		then CEILING(DATEDIFF(DAY, Coalesce(p.startdate,EH.entstartdate), coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) >EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 0-- Start late only and Active
		then CEILING(DATEDIFF(DAY,p.startdate, coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 1 -- Started and Now Inactive
		then CEILING(DATEDIFF(DAY,EH.entstartdate, coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 0 then 
			case when EH.defaultent = 0 -- give full amount if joined full year
				then EH.entamount 
					else coalesce(ED.entamount,0)
						END +coalesce(ED.ContractedHoliday,0) 
	end +coalesce(EH.broughtfwd,0)+[dbo].[fnEmployeeLSA](EH.empref,EH.sysyear,case when EH.HolGroupRef = -1 then 1 else EH.HolGroupRef end)+ coalesce(BankHolsWorked.[N.W.Day get only worked hrs],BankHolsWorked.CountDate,0)
						-coalesce(AbsBookedTotal.AbsTotal,0) 
					,2)
			as varchar(10))
	END	as [Errors],


	case when ED.ContractedHoliday>0 then 'Contractual' Else 'Statutory' End as [Check],  -- Now calculated totals
	case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0) as CalcAmount,
	Eh.broughtfwd as [DB Brt fwd2],
	[dbo].[fnEmployeeLSA](EH.empref,EH.sysyear,case when EH.HolGroupRef = -1 then 1 else EH.HolGroupRef end) as LSA,
	coalesce(BankHolsWorked.CountDate,0) as [Calc Worked BH],
	coalesce(BankHolsWorked.[N.W.Day get only worked hrs],BankHolsWorked.CountDate,0) as [Calc BH Hrs/Days],
	
	
	case 
		when coalesce(NullIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entstartdate) < EH.entstartdate then 0
		when Coalesce(p.startdate,EH.entstartdate) >EH.entstartdate and dbo.fnIsLeaver (EH.empref) = 1-- Start late Finished Early
		then CEILING(DATEDIFF(DAY, Coalesce(p.startdate,EH.entstartdate), coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) >EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 0-- Start late only and Active
		then CEILING(DATEDIFF(DAY,p.startdate, coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 1 -- Started and Now Inactive
		then CEILING(DATEDIFF(DAY,EH.entstartdate, coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 0 then 
			case when EH.defaultent = 0 -- give full amount if joined full year
				then EH.entamount 
					else coalesce(ED.entamount,0)
						END +coalesce(ED.ContractedHoliday,0) 
	end as [CalcProRota],
	
	
	--case when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate then case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0) else -- give full amount if joined full year
	--	round(Round((   DATEDIFF(DAY, Coalesce(p.startdate,EH.entstartdate), EH.entenddate) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)-(select count(globalref) from globabs where absdate < Getdate())) / 365)/5, 1)*5 ,1) end as [CalcProRota2],
	--case when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate then case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)+coalesce(EH.broughtfwd,0) else -- give full amount if joined full year
	--	CEILING(    DATEDIFF(DAY, Coalesce(p.startdate,EH.entstartdate), EH.entEnddate) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365+coalesce(EH.broughtfwd,0)) end as [CalcProRota+BF],

	case 
		when coalesce(NullIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entstartdate) < EH.entstartdate then 0
		when Coalesce(p.startdate,EH.entstartdate) >EH.entstartdate and dbo.fnIsLeaver (EH.empref) = 1-- Start late Finished Early
		then CEILING(DATEDIFF(DAY, Coalesce(p.startdate,EH.entstartdate), coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) >EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 0-- Start late only and Active
		then CEILING(DATEDIFF(DAY,p.startdate, coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 1 -- Started and Now Inactive
		then CEILING(DATEDIFF(DAY,EH.entstartdate, coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 0 then 
			case when EH.defaultent = 0 -- give full amount if joined full year
				then EH.entamount 
					else coalesce(ED.entamount,0)
						END +coalesce(ED.ContractedHoliday,0) 
		end +coalesce(EH.broughtfwd,0) as [CalcProRota+BF],

	case
		when coalesce(NullIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entstartdate) < EH.entstartdate then 0
		when Coalesce(p.startdate,EH.entstartdate) >EH.entstartdate and dbo.fnIsLeaver (EH.empref) = 1-- Start late Finished Early
		then CEILING(DATEDIFF(DAY, Coalesce(p.startdate,EH.entstartdate), coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) >EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 0-- Start late only and Active
		then CEILING(DATEDIFF(DAY,p.startdate, coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 1 -- Started and Now Inactive
		then CEILING(DATEDIFF(DAY,EH.entstartdate, coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 0 then 
			case when EH.defaultent = 0 -- give full amount if joined full year
				then EH.entamount 
					else coalesce(ED.entamount,0)
						END +coalesce(ED.ContractedHoliday,0) 
	end +coalesce(EH.broughtfwd,0)+[dbo].[fnEmployeeLSA](EH.empref,EH.sysyear,case when EH.HolGroupRef = -1 then 1 else EH.HolGroupRef end) as [CalcProRota+BF+LSA],
	case 
		when coalesce(NullIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entstartdate) < EH.entstartdate then 0
		when Coalesce(p.startdate,EH.entstartdate) >EH.entstartdate and dbo.fnIsLeaver (EH.empref) = 1-- Start late Finished Early
		then CEILING(DATEDIFF(DAY, Coalesce(p.startdate,EH.entstartdate), coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) >EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 0-- Start late only and Active
		then CEILING(DATEDIFF(DAY,p.startdate, coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 1 -- Started and Now Inactive
		then CEILING(DATEDIFF(DAY,EH.entstartdate, coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 0 then 
			case when EH.defaultent = 0 -- give full amount if joined full year
				then EH.entamount 
					else coalesce(ED.entamount,0)
						END +coalesce(ED.ContractedHoliday,0) 
	end +coalesce(EH.broughtfwd,0)+[dbo].[fnEmployeeLSA](EH.empref,EH.sysyear,case when EH.HolGroupRef = -1 then 1 else EH.HolGroupRef end)+ coalesce(BankHolsWorked.[N.W.Day get only worked hrs],BankHolsWorked.CountDate,0) as [CalcProRota+BF+LSA+BH],
	dbo.fnCurrentEntitlement(EH.empref,EH.sysyear,EH.groupref) as RTamount,
	Case when ED.enttype = 1 then 'Hour =' else 'Days =' End as [Type2],
	coalesce(AbsBookedTotal.AbsTotal,0) as AbsTotal,
	coalesce(Takensum.TakenDays,0) as [Calc Taken],
	Case when ED.enttype = 1 then 'Hour =' else 'Days =' End as [Type3],

			round
				(	case 
		when coalesce(NullIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entstartdate) < EH.entstartdate then 0
		when Coalesce(p.startdate,EH.entstartdate) >EH.entstartdate and dbo.fnIsLeaver (EH.empref) = 1-- Start late Finished Early
		then CEILING(DATEDIFF(DAY, Coalesce(p.startdate,EH.entstartdate), coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) >EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 0-- Start late only and Active
		then CEILING(DATEDIFF(DAY,p.startdate, coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 1 -- Started and Now Inactive
		then CEILING(DATEDIFF(DAY,EH.entstartdate, coalesce(NULLIF(P.enddate,'1899-12-30 00:00:00.000'),EH.entenddate)) * (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0)) / 365)
		when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate and  dbo.fnIsLeaver (EH.empref) = 0 then 
			case when EH.defaultent = 0 -- give full amount if joined full year
				then EH.entamount 
					else coalesce(ED.entamount,0)
						END +coalesce(ED.ContractedHoliday,0) 
	end +coalesce(EH.broughtfwd,0)+[dbo].[fnEmployeeLSA](EH.empref,EH.sysyear,case when EH.HolGroupRef = -1 then 1 else EH.HolGroupRef end)+ coalesce(BankHolsWorked.[N.W.Day get only worked hrs],BankHolsWorked.CountDate,0)
						-coalesce(AbsBookedTotal.AbsTotal,0) 
					,2) as  [Balance Hrs or Days]

			--round
			--	(case 
			--		when EH.defaultent = 0 
			--			then coalesce(Eh.entamount,ED.entamount,0)  +EH.broughtfwd +[dbo].[fnEmployeeLSA](EH.empref,EH.sysyear,case when EH.HolGroupRef = -1 then 1 else EH.HolGroupRef end) 
			--			+ coalesce(BankHolsWorked.[N.W.Day get only worked hrs],BankHolsWorked.CountDate,0)
			--			-coalesce(AbsBookedTotal.AbsTotal,0)
			--		when Coalesce(p.startdate,EH.entstartdate) <=EH.entstartdate then (case when EH.defaultent = 0 then EH.entamount else coalesce(ED.entamount,0)END +coalesce(ED.ContractedHoliday,0))+coalesce(EH.broughtfwd,0) 
			--			+ coalesce(BankHolsWorked.[N.W.Day get only worked hrs],BankHolsWorked.CountDate,0) +[dbo].[fnEmployeeLSA](EH.empref,EH.sysyear,case when EH.HolGroupRef = -1 then 1 else EH.HolGroupRef end)
			--			-coalesce(AbsBookedTotal.AbsTotal,0) 
			--		else 
			--			CEILING(DATEDIFF(DAY, Coalesce(p.startdate,EH.entstartdate), EH.entenddate) * (case when EH.defaultent = 0 then coalesce(Eh.entamount,ED.entamount,0) + coalesce(ED.ContractedHoliday,0) else coalesce(ED.entamount,0) + coalesce(ED.ContractedHoliday,0) End) / 365) +EH.broughtfwd 
			--			+ coalesce(BankHolsWorked.[N.W.Day get only worked hrs],BankHolsWorked.CountDate,0) + [dbo].[fnEmployeeLSA](EH.empref,EH.sysyear,case when EH.HolGroupRef = -1 then 1 else EH.HolGroupRef end)
			--			-coalesce(AbsBookedTotal.AbsTotal,0) 
			--		END,2) as  [Balance Hrs or Days]

	,EMP.WebLogin
	,dbo.fnDecode (EMP.WebPassword) as WebPass
	,EH.entref
	,ED.enttype

into #EntitlementsTemp
-- compare All Holiday Groups with different start dates & Full join to highlight if any Empref in Entitlements table that no longer exist.
from entitlements EH
--Full join absences A on A.empref = EH.empref
Full join empdetails EMP on EMP.empref = EH.empref
Full join personnel P on P.empref = EH.empref
Full join absgroups AG on AG.groupref = Eh.HolGroupRef
--Full join abscodes Ab on Ab.coderef = A.coderef
full join entitlementdefault ED on ED.groupref = EMP.HolEntGroupRef--coalesce(NullIF(EH.HolGroupRef,-1),1)

Outer Apply (select Empref as EMP2,
					Case when ED.enttype = 1 then 
						round(CAST(SUM(dbo.fnGetAbsHours(A1.absref)/3600.0) AS float),2)
					else -- give full amount if joined full year
						(sum(case when A1.Abstype=1 then 1.0 when A1.Abstype=2 then 0.5 when A1.Abstype=3 then 0.5 end)) 
					End as TakenDays
				from absences A1 
				Full join abscodes Ab3 on Ab3.coderef = A1.coderef
				where Absdate between EH.entstartdate and Getdate() and A1.empref = EH.empref and Ab3.groupref = 1
				group by A1.empref) as TakenSum 

Outer Apply (select Empref as EMP2,
					Case when ED.enttype = 1 then 
						round(CAST(SUM(dbo.fnGetAbsHours(A1.absref)/3600.0) AS float),2)
					else -- give full amount if joined full year
						(sum(case when A1.Abstype=1 then 1.0 when A1.Abstype=2 then 0.5 when A1.Abstype=3 then 0.5 when A1.Abstype IS NULL then 0.0 end)) 
					End as AbsTotal
				from absences A1 
				Full join abscodes Ab3 on Ab3.coderef = A1.coderef
				where Absdate between EH.entstartdate and EH.entenddate and A1.empref = EH.empref and Ab3.groupref = 1
				group by A1.empref) as AbsBookedTotal 

Outer Apply ( 
				select 
				CL.empref, 
				--CL.clockdate,
				count(cl.empref) as [BH and Worked],
				Case when @additionalBKHent = 1 then count(distinct cl.clockdate) else 0 END As CountDate,
/*These will check if staff should be working on B hol or marked as Day Off. 
If day off then they only get hours worked as not considered a bank holiday
else they get the target amount as working on a B hol.
N.W.Day get only worked hrs will return NULL if not met and then we use CountDate to give distinct amount of days
This prevents clock used 4 times and generating 4 bank holidays but leave hours as 1
*/
				Case when @additionalBKHent = 1 then
					Cast(Sum
							(Case when dbo.fnFindDefaultShiftForEmp(cl.EmpRef,CL.clockdate) = -1 then (CR.rate1)/(60.00*60)
								else
									Case when enttype=1 then dbo.fnGetShiftExpectedHours(cl.empref,cl.ClockDate,cl.clockingtimesref) /(60.00*60) 
										else NULL end
							END)
							--	/Case when enttype=1 then 1 else count(distinct cl.clockdate) END-- This prevents clock used 4 times and generating 4 bank holidays but leave hours as 1
							as decimal(10,1)) 
				ELSE 0 END
					 as [N.W.Day get only worked hrs]


				from clockingtimes CL
				join globabs GA on GA.absdate = CL.clockdate
				join entitlements E on E.empref = CL.empref and groupref = 1
				--join empdetails ED on ED.empref = cl.empref
				join clockingrates CR on CR.empref = cl.empref and CR.clockdate = GA.absdate
				where GA.absdate = CL.clockdate and e.empref = EH.empref
				group by CL.empref,E.AdditionalEnt,E.enttype) as BankHolsWorked

				outer apply (select E4.entref as Entref2 from entitlements E4 where groupref = eh.holgroupref and E4.empref = EH.empref)  as DupGroup

				outer apply (select top 1 EHD.Empref,EHD.endDate from EmpHisDates EHD
								where EHD.Empref = EH.empref 
								and EHD.endDate between EH.entstartdate and EH.entenddate
								) as Ternimated	

				outer apply(select top 1
								b.empref,
								b.absdate
								--count (B.empref)
								from absences B
										join entitlements E on E.empref = B.empref and E.groupref = 1
										join abscodes Ab on Ab.coderef = B.coderef and AB.groupref = 1 -- all holidays
										where absdate between e.entstartdate and e.entenddate and b.empref = eh.empref
								group by absdate,abstype,E.empref,B.empref,b.absdate
								having (count (B.empref)> 1)) DupCheck
				outer apply (  select top 1
									 EMP.empref,
									 S.name as [Shift Name]
									from shiftabs sh
									full join Shifts S on SH.shiftref = S.shiftref
									left join Ratesnames R on R.rateno = sh.ratesref
									left join scheduleshift SC on SC.shiftref = sh.shiftref
									left join empdetails EMP on EMP.scheduleref = SC.scheduleref
									where sh.ratesref = 0 and EMP.empref = EH.empref) Unpaid 

-- End of Outer Apply

where 
EH.groupref = 1
																		-- also captures if has more than 1 Holiday Group
--and (P.enddate = '1899-12-30 00:00:00.000' or P.enddate IS NULL or P.enddate>getdate())	-- Excludes leaves
--and (BankHolsWorked.[N.W.Day get only worked hrs]>0 or BankHolsWorked.CountDate>0)
group by EH.empref,EMP.forenames,EMP.surname,EH.HolGroupRef,EH.calabstaken,Eh.broughtfwd,EH.entamount,EH.balance,p.startdate,AG.groupname,EH.LSAEntAmount,EH.absbooked,ED.entamount,ED.ContractedHoliday,ed.enttype,EH.entstartdate,EH.entenddate,EMP.Payrollno
,EH.sysyear,EH.enttype,P.endDate,EH.additionalent,Takensum.TakenDays,BankHolsWorked.[N.W.Day get only worked hrs],BankHolsWorked.CountDate,AbsBookedTotal.AbsTotal,EH.entref,Eh.defaultent,EMP.HolEntGroupRef,ED.entstartdate,EMP.WebLogin,EMP.WebPassword,EH.abstaken
,DupCheck.empref,Ternimated.empref,Ternimated.endDate,DupCheck.absdate,EH.groupref,DupGroup.Entref2,Unpaid.empref,Unpaid.[Shift Name]



select * from #EntitlementsTemp 
--where Warnings like 'Start%'
--where DBBalance <0
--where Errors like 'Duplicate Holidays'
order by forenames

----Do next part seperately 
--DECLARE @SQL nvarchar(2000)
--SET @SQL='select *
--INTO dbo.Entitlements_'+REPLACE(CONVERT(nvarchar(50),GETDATE(),101),'/','')+'
--from dbo.Entitlements'
--EXEC(@SQL)
--GO

--Update E
--set 
--entamount = case when ED.ContractedHoliday > 0 and coalesce(E.entamount,0) = 0 then 0 else calcamount end,
--absbooked = [AbsTotal],
--balance = [Balance Hrs or Days],
--LSAEntAmount=LSA,
--calabstaken =[Calc Taken],
--AdditionalEnt = ET.[Calc BH Hrs/Days],
--enttype = ET.enttype
--from entitlements E
--join #EntitlementsTemp ET on ET.entref = E.entref
--join entitlementdefault ED on ED.groupref = Case when E.HolGroupRef = -1 then 1 else E.HolGroupRef end
--where ET.entref = E.entref and E.groupref = 1 --and E.defaultent = 1
