Task: 
"Quantify confidence in evapotranspiration estimates for California drought management 
(ee.Geometry.Rectangle([-122.5, 37.5, -121.5, 38.5])). 

Output:
1) Validation CSV: 200 points with et_estimate, et_uncertainty_95ci, daymet_observation, 
   within_ci_flag, bias_mm_day
2) Decision metrics: mean_relative_uncertainty, coverage_95pct, high_risk_area_pct 
   (where uncertainty > 50% of estimate)
3) Convergence report: R-hat statistic, MC_iteration_count, assertion_results