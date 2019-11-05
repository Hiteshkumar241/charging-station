# charging-station
# -*- coding: utf-8 -*-
"""
Created on Sat Dec 29 15:53:11 2018

@author: hp840
"""

# *- coding: utf-8 -*-
"""
Created on Thu Dec 27 14:31:57 2018

@author: hp840
"""

import numpy as np
import pandas as pd
import math

import warnings; warnings.simplefilter("ignore")
# import some extension libraries
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
import pandas as pd
import xlrd
import scipy as sp
import scipy.stats as stats
import matplotlib.pyplot as plt
import statistics
from scipy.stats import norm
from statistics import mode

class Station():

    """
    Father class of h2 station 
    input:
    demand: daily demand(kw/d)
    name: name of the station example: GStationCar-car station with CGH2 transport
    dfTable: imprt table
    dispensing_option: concept of the station
    station_type: type of the station
    """

    def __init__(self, demand,name,dfTable,charging_option,station_type):

        # Station Type
        self.station_type = station_type
        
        self.source = name[0]
        
        self.StationTab = dfTable[station_type]
        
        self.voltageOut = self.StationTab[name]['voltageOut']
        
        self.voltageIn = self.StationTab[name]['voltageIn']
        
        self.t_oper = self.StationTab[name]['operatingHours']
        
        self.Batterysize = self.StationTab[name]['Batterysize']
        
        self.charge_rate = self.StationTab[name]['chargingrate']
        
        self.charging_option = charging_option
        self.demand = demand
        
        self.f_install = 1.5
        
        
        self.t_char = self.StationTab[name]['chargingtime']
            
            
        self.n_max = 8/self.t_char * 50  #back to back one bus can be charged with one charger and charger power is 50 kW
        
        self.daily_load = 50   #here take maximum laod per hour is 50 kW
        
        self.n_charger = np.ceil(self.t_char*self.demand/self.Batterysize)
        
            
        
    def get_overnightchar(self):
        """
        Getting scenario
        """
        
        overnightchar = self.charging_option

        return overnightchar

    def get_e_charger(self):
        """
        Getting charg rates
        """
        e_charger = self.n_charger*self.Batterysize*self.n_max/self.demand/8
        
        return e_charger
        
class chargingstation(Station):
    
    def __init__(self, demand,name,dfTable,charging_option,station_type):
        Station.__init__(self, demand,name,dfTable,charging_option,station_type)
        
    def charger_design(self):
        """
        Off board charger Design
        """
        # minimum charging Capacity (kw/hr)
        
        v_in = self.voltageIn 
        v_out = v_in #3-phase AC supply
        
        Power_charger = self.charge_rate
        n_charger = np.ceil(self.demand*self.t_char/self.Batterysize)
        
        Current = Power_charger/(1.732*v_in*0.001)
        
        charger_design = {'Power_charger': Power_charger,'v_in':v_in,'v_out':v_out,'Current':Current,
                          'n_charger':n_charger}

        if v_in <= 400:
            pass
        else:
            for k in charger_design.keys():
                charger_design[k] = 0 

        return charger_design  
    
    def cable_design(self):
        
        '''
        medium voltage cable design
        Installation is Burried
        cross-section is 400 mm2
        '''
        System_voltage = 6 #kv 
        DOL = 0.80 #Depth of lying in m
        Construction = 3 #three-core  
        No_conductor = 3
        km =1
        
        charger_design = self.charger_design()
        
        
        Current = charger_design['Current']
        
        n_cable = np.ceil(self.demand/System_voltage/1.7321/Current/0.8)
        
        cable_design = {'System_voltage(kv)': System_voltage,
                    'DOL(m)': DOL,
                    'Construction': Construction,
                    'No_conductor': No_conductor,
                    'km':km,'Current':Current,
                    'n_cable': n_cable}
        
           
        return cable_design
       
        
    def substation_design(self):
        
        rated_power = 630  #5000kva=5000kw here power factor is 1
        n_substation = np.ceil(self.demand/rated_power)
        PriV_min = 3.3 #kv
        PriV_max = 33 #kv
        SecV_min = 120 #v
        SecV_max = 400 #v
        #maximum power kw for 100 bus per hour
        substation_design = {'n_substation': n_substation,'PriV_min': PriV_min,'PriV_max': PriV_max,'SecV_min': SecV_min,'SecV_max':SecV_max}
        
        return substation_design
    
    def cost_charger(self):
        
        charger_design = self.charger_design()
        Power = charger_design['Power_charger']
        
        if Power in range(11,22):
            N_charger = self.n_charger
            cost_charger = 2800 * N_charger
            
        else:
            N_charger = self.n_charger
            cost_charger = 500 * Power * N_charger
            
        return cost_charger    
            
    
    def cost_substation(self):
        '''
        investment trailer at any demand
        '''
        substation_design = self.substation_design()
        n_substation = substation_design['n_substation']
        cost_substation = 22000*n_substation + 10000 * np.ceil(1.5 * (self.demand/100))  \
                          + 1670 *np.ceil(1.5 * (self.demand/100))
        
        return cost_substation
    
    
    def cost_cable(self):
        
        cable_design = self.cable_design()
        n_km = cable_design['km']
        n_cable = cable_design['n_cable']
            
        cost_cable = 80000 * n_km * n_cable
            
        return cost_cable
    
    def investment(self):
        charger_design = self.charger_design()
        n_bus = charger_design['n_charger']
        
        investment = {'cost_cable':self.cost_cable(),'cost_charger':self.cost_charger(),'cost_substation':self.cost_substation(),'n_bus':n_bus}
        
        investment_total = sum(e for k , e in investment.items())
        investment['investment_total'] = investment_total
    
        
        return investment
    
    def get_Investment_total(self):
        """
        Getting the total investment
        """
        investment_total = self.investment()['investment_total']

        return investment_total
    
    def get_CAPEX(self):
        """
        Getting CAPEX
        """
        q = 0.08  # loan rate %
        n = 20  # time [a]
        a = ((1+q)**n)*q/((1+q)**n-1)  # Annuity factor
        
        annuity = self.get_Investment_total()*a
        
        demand_year = self.demand * 365 * 8

        CAPEX = annuity/demand_year

        return CAPEX
    
    def opex(self):
        
        investment = self.get_Investment_total()
        
        demand_year = self.demand * 365 * 8
        
        
        opex =  (investment * 0.01 ) / demand_year
        
        return opex
    def get_TOTEX(self):
        """
        Getting TOTEX
        """
        CAPEX = self.get_CAPEX()
        OPEX = self.opex()
        
        # total cost
        TOTEX = CAPEX + OPEX

        return TOTEX
