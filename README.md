# FUTURE_FS_02
Build a WEATHER DASHBOARD

# Main Weather Dashboard Page
// client/src/pages/weather-dashboard.tsx

import React, { useState, useEffect } from "react";
import { useQuery } from "@tanstack/react-query";
import { CloudSun } from "lucide-react";
import { SearchBar } from "@/components/weather/search-bar";
import { CurrentWeatherCard } from "@/components/weather/current-weather-card";
import { ForecastCard } from "@/components/weather/forecast-card";
import { HourlyForecast } from "@/components/weather/hourly-forecast";
import { FavoritesSidebar } from "@/components/weather/favorites-sidebar";
import { WeatherInsights } from "@/components/weather/weather-insights";
import { weatherApi } from "@/lib/weather-api";
import { useToast } from "@/hooks/use-toast";
import type { CitySearchResult, WeatherData, ForecastData, DailyForecast, HourlyForecast as HourlyForecastType } from "@/types/weather";

export default function WeatherDashboard() {
  const [selectedCity, setSelectedCity] = useState<CitySearchResult | null>(null);
  const { toast } = useToast();

  // Set initial city to Hyderabad
  useEffect(() => {
    // Set Hyderabad as the initial city
    setSelectedCity({
      name: "Hyderabad",
      lat: 17.3850,
      lon: 78.4867,
      country: "IN",
    });
    
    // Show message about the initial city
    toast({
      title: "Welcome to Weather Dashboard",
      description: "Showing weather for Hyderabad, India",
    });
  }, []);

  // Fetch current weather data
  const { 
    data: currentWeather, 
    isLoading: isWeatherLoading, 
    error: weatherError 
  } = useQuery({
    queryKey: ["/api/weather/current", selectedCity?.lat, selectedCity?.lon],
    queryFn: () => 
      selectedCity 
        ? weatherApi.getCurrentWeatherByCoords(selectedCity.lat, selectedCity.lon)
        : Promise.reject("No city selected"),
    enabled: !!selectedCity,
    retry: 1,
  });

  // Handle other functions and rendering
  // ...
}

# Current Weather Card Component
// client/src/components/weather/current-weather-card.tsx

import { Card, CardContent } from "@/components/ui/card";
import { getWeatherIconUrl, formatTemp } from "@/lib/weather-api";
import type { WeatherData } from "@/types/weather";

export function CurrentWeatherCard({ weather, isLoading }: CurrentWeatherCardProps) {
  if (isLoading) {
    return <LoadingState />;
  }

  const getGradientClass = (condition: string) => {
    // Dynamic gradient backgrounds based on weather conditions
    const mainCondition = condition.toLowerCase();
    if (mainCondition.includes('clear') || mainCondition.includes('sunny')) {
      return "bg-gradient-to-br from-pink-400 via-red-400 to-yellow-400";
    } else if (mainCondition.includes('cloud')) {
      return "bg-gradient-to-br from-blue-400 via-cyan-400 to-teal-400";
    } else if (mainCondition.includes('rain') || mainCondition.includes('drizzle')) {
      return "bg-gradient-to-br from-green-400 via-cyan-400 to-blue-400";
    } else {
      return "bg-gradient-to-br from-purple-400 via-indigo-500 to-blue-600";
    }
  };

  return (
    <Card className={`${getGradientClass(weather.weather[0].main)} text-white relative overflow-hidden`}>
      {/* Weather card content with temperature, conditions, etc. */}
    </Card>
  );
}

# Backend Code
API Routes

// server/routes.ts

import type { Express } from "express";
import { createServer, type Server } from "http";
import { storage } from "./storage";
import { insertFavoriteCitySchema } from "@shared/schema";

export async function registerRoutes(app: Express): Promise<Server> {
  // Get current weather data
  app.get("/api/weather/current", async (req, res) => {
    try {
      const { lat, lon, q } = req.query;
      
      // Hardcoded data for Hyderabad
      if (q === 'Hyderabad' || (lat === '17.385' && lon === '78.4867')) {
        return res.json({
          "coord": {"lon": 78.4867, "lat": 17.385},
          "weather": [{"id": 800, "main": "Clear", "description": "clear sky", "icon": "01d"}],
          "base": "stations",
          "main": {
            "temp": 33.2,
            "feels_like": 31.9,
            "temp_min": 33.2,
            "temp_max": 33.2,
            "pressure": 1009,
            "humidity": 28
          },
          // Other weather data
        });
      }

      // Actual API call using OpenWeather API
      let url = `https://api.openweathermap.org/data/2.5/weather?units=metric`;
      // ... rest of API handling
    } catch (error) {
      console.error("Weather API error:", error);
      res.status(500).json({ error: "Failed to fetch weather data" });
    }
  });

  // Other routes: forecast, search, favorites
  // ...
}

# Database Schema

// shared/schema.ts

import { pgTable, text, serial, timestamp, decimal } from "drizzle-orm/pg-core";
import { createInsertSchema } from "drizzle-zod";
import { z } from "zod";

export const favoriteCities = pgTable("favorite_cities", {
  id: serial("id").primaryKey(),
  name: text("name").notNull(),
  country: text("country").notNull(),
  lat: decimal("lat", { precision: 10, scale: 7 }).notNull(),
  lon: decimal("lon", { precision: 10, scale: 7 }).notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});

export const insertFavoriteCitySchema = createInsertSchema(favoriteCities).omit({
  id: true,
  createdAt: true,
});

export type InsertFavoriteCity = z.infer<typeof insertFavoriteCitySchema>;
export type FavoriteCity = typeof favoriteCities.$inferSelect;

# Storage Implementation
// server/storage.ts

import { favoriteCities, type FavoriteCity, type InsertFavoriteCity } from "@shared/schema";

export interface IStorage {
  getFavoriteCities(): Promise<FavoriteCity[]>;
  addFavoriteCity(city: InsertFavoriteCity): Promise<FavoriteCity>;
  removeFavoriteCity(id: number): Promise<void>;
  getFavoriteCityByCoords(lat: number, lon: number): Promise<FavoriteCity | undefined>;
}

export class MemStorage implements IStorage {
  private cities: Map<number, FavoriteCity>;
  private currentId: number;

  constructor() {
    this.cities = new Map();
    this.currentId = 1;
  }

  // Implementation of storage methods
  // ...
}

export const storage = new MemStorage();
