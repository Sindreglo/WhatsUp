# Fix: Plan Activation Error with Supabase

## Problem

When trying to activate a workout plan, you get the error:

```
❌ Failed to fetch plan content from Supabase: The data couldn't be read because it isn't in the correct format.
```

This is a Swift `JSONDecoder` error caused by a mismatch between your data models and the actual JSON structure from Supabase.

## Root Causes

Based on the workout_plans data from Supabase, there are two issues:

### 1. The `rating` field is a String, not a Double

In Supabase, the `rating` field is stored as:
```json
"rating": "4.90"
```

But if your Swift model expects a `Double`:
```swift
// ❌ This will fail
let rating: Double
```

### 2. The `content` field is a JSON String, not a JSON Object

The `content` field contains a stringified JSON:
```json
"content": "{\"weeks\": [{\"note\": \"Week 1...\", \"sessions\": [...]}]}"
```

This needs to be decoded in two steps: first as a String, then parsed as JSON.

## Solution

Update your Swift models to handle these data types correctly:

### WorkoutPlan Model

```swift
import Foundation

// MARK: - WorkoutPlan
struct WorkoutPlan: Codable, Identifiable {
    let idx: Int
    let id: Int
    let title: String
    let author: String
    let verified: Bool
    let price: Int
    let rating: String  // ✅ Changed from Double to String
    let downloads: Int
    let tags: [String]
    let description: String
    let difficulty: String
    let weeksCount: Int
    let sessionsPerWeek: Int
    let content: String  // ✅ This is a JSON string, not an object
    let createdAt: String
    let updatedAt: String
    
    enum CodingKeys: String, CodingKey {
        case idx, id, title, author, verified, price, rating, downloads, tags, description, difficulty
        case weeksCount = "weeks_count"
        case sessionsPerWeek = "sessions_per_week"
        case content
        case createdAt = "created_at"
        case updatedAt = "updated_at"
    }
    
    // Helper to get rating as Double
    var ratingValue: Double {
        return Double(rating) ?? 0.0
    }
    
    // Helper to decode content JSON string
    func decodedContent() throws -> PlanContent {
        guard let data = content.data(using: .utf8) else {
            throw DecodingError.dataCorrupted(
                DecodingError.Context(codingPath: [], debugDescription: "Could not convert content string to data")
            )
        }
        return try JSONDecoder().decode(PlanContent.self, from: data)
    }
}

// MARK: - PlanContent (decoded from the content string)
struct PlanContent: Codable {
    let weeks: [Week]
}

struct Week: Codable, Identifiable {
    // Note: Using UUID() generates a new ID each decode. For SwiftUI lists,
    // consider using a stable identifier from your data or caching decoded results.
    let id: UUID
    let note: String
    let sessions: [Session]
    
    enum CodingKeys: String, CodingKey {
        case note, sessions
    }
    
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        // Generate stable ID based on note content (or use index-based approach)
        self.id = UUID()
        self.note = try container.decode(String.self, forKey: .note)
        self.sessions = try container.decode([Session].self, forKey: .sessions)
    }
}

struct Session: Codable, Identifiable {
    // Note: Using UUID() generates a new ID each decode. For SwiftUI lists,
    // consider using a stable identifier from your data or caching decoded results.
    let id: UUID
    let name: String
    let type: String
    let exercises: [Exercise]
    
    enum CodingKeys: String, CodingKey {
        case name, type, exercises
    }
    
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        // Generate stable ID based on session name (or use index-based approach)
        self.id = UUID()
        self.name = try container.decode(String.self, forKey: .name)
        self.type = try container.decode(String.self, forKey: .type)
        self.exercises = try container.decode([Exercise].self, forKey: .exercises)
    }
}

struct Exercise: Codable, Identifiable {
    let id: String  // UUID string from Supabase
    let name: String
    let note: String
    let reps: String
    let sets: String
    let weight: String
}
```

### Usage Example

```swift
// Fetching and decoding the plan
func fetchPlan(planId: Int) async throws -> WorkoutPlan {
    let response = try await supabase
        .from("workout_plans")
        .select()
        .eq("id", value: planId)
        .single()
        .execute()
    
    let decoder = JSONDecoder()
    let plan = try decoder.decode(WorkoutPlan.self, from: response.data)
    return plan
}

// Activating the plan (decoding the content)
func activatePlan(_ plan: WorkoutPlan) async throws {
    do {
        let content = try plan.decodedContent()
        // Now you have access to content.weeks, each week's sessions, etc.
        print("Plan has \(content.weeks.count) weeks")
        
        for week in content.weeks {
            print("Week note: \(week.note)")
            for session in week.sessions {
                print("  Session: \(session.name) - \(session.exercises.count) exercises")
            }
        }
    } catch {
        print("❌ Failed to decode plan content: \(error)")
        throw error
    }
}
```

### Alternative: Custom Decoder

If you want the content to be automatically decoded, use a custom decoder:

```swift
struct WorkoutPlan: Codable, Identifiable {
    let idx: Int
    let id: Int
    let title: String
    let author: String
    let verified: Bool
    let price: Int
    let rating: String
    let downloads: Int
    let tags: [String]
    let description: String
    let difficulty: String
    let weeksCount: Int
    let sessionsPerWeek: Int
    let content: PlanContent  // ✅ Decoded automatically
    let createdAt: String
    let updatedAt: String
    
    // Helper to get rating as Double
    var ratingValue: Double {
        return Double(rating) ?? 0.0
    }
    
    enum CodingKeys: String, CodingKey {
        case idx, id, title, author, verified, price, rating, downloads, tags, description, difficulty
        case weeksCount = "weeks_count"
        case sessionsPerWeek = "sessions_per_week"
        case content
        case createdAt = "created_at"
        case updatedAt = "updated_at"
    }
    
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        
        idx = try container.decode(Int.self, forKey: .idx)
        id = try container.decode(Int.self, forKey: .id)
        title = try container.decode(String.self, forKey: .title)
        author = try container.decode(String.self, forKey: .author)
        verified = try container.decode(Bool.self, forKey: .verified)
        price = try container.decode(Int.self, forKey: .price)
        rating = try container.decode(String.self, forKey: .rating)
        downloads = try container.decode(Int.self, forKey: .downloads)
        tags = try container.decode([String].self, forKey: .tags)
        description = try container.decode(String.self, forKey: .description)
        difficulty = try container.decode(String.self, forKey: .difficulty)
        weeksCount = try container.decode(Int.self, forKey: .weeksCount)
        sessionsPerWeek = try container.decode(Int.self, forKey: .sessionsPerWeek)
        createdAt = try container.decode(String.self, forKey: .createdAt)
        updatedAt = try container.decode(String.self, forKey: .updatedAt)
        
        // Decode content from JSON string
        let contentString = try container.decode(String.self, forKey: .content)
        guard let contentData = contentString.data(using: .utf8) else {
            throw DecodingError.dataCorrupted(
                DecodingError.Context(codingPath: [CodingKeys.content], debugDescription: "Could not convert content string to data")
            )
        }
        content = try JSONDecoder().decode(PlanContent.self, from: contentData)
    }
    
    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        
        try container.encode(idx, forKey: .idx)
        try container.encode(id, forKey: .id)
        try container.encode(title, forKey: .title)
        try container.encode(author, forKey: .author)
        try container.encode(verified, forKey: .verified)
        try container.encode(price, forKey: .price)
        try container.encode(rating, forKey: .rating)
        try container.encode(downloads, forKey: .downloads)
        try container.encode(tags, forKey: .tags)
        try container.encode(description, forKey: .description)
        try container.encode(difficulty, forKey: .difficulty)
        try container.encode(weeksCount, forKey: .weeksCount)
        try container.encode(sessionsPerWeek, forKey: .sessionsPerWeek)
        try container.encode(createdAt, forKey: .createdAt)
        try container.encode(updatedAt, forKey: .updatedAt)
        
        // Encode content back to JSON string
        let contentData = try JSONEncoder().encode(content)
        guard let contentString = String(data: contentData, encoding: .utf8) else {
            throw EncodingError.invalidValue(content, 
                EncodingError.Context(codingPath: [CodingKeys.content], debugDescription: "Could not convert content data to string"))
        }
        try container.encode(contentString, forKey: .content)
    }
}
```

## Debugging Tips

If you're still having issues, add this debugging code to see exactly what's failing:

```swift
func fetchAndDebugPlan(planId: Int) async {
    do {
        let response = try await supabase
            .from("workout_plans")
            .select()
            .eq("id", value: planId)
            .single()
            .execute()
        
        // Print raw JSON to see structure
        if let jsonString = String(data: response.data, encoding: .utf8) {
            print("Raw JSON: \(jsonString)")
        }
        
        // Try to decode
        let decoder = JSONDecoder()
        let plan = try decoder.decode(WorkoutPlan.self, from: response.data)
        print("✅ Successfully decoded plan: \(plan.title)")
        
    } catch let DecodingError.keyNotFound(key, context) {
        print("❌ Missing key '\(key.stringValue)' – \(context.debugDescription)")
        print("Path: \(context.codingPath.map { $0.stringValue })")
    } catch let DecodingError.typeMismatch(type, context) {
        print("❌ Type mismatch for \(type) – \(context.debugDescription)")
        print("Path: \(context.codingPath.map { $0.stringValue })")
    } catch let DecodingError.valueNotFound(type, context) {
        print("❌ Value not found for \(type) – \(context.debugDescription)")
        print("Path: \(context.codingPath.map { $0.stringValue })")
    } catch let DecodingError.dataCorrupted(context) {
        print("❌ Data corrupted – \(context.debugDescription)")
        print("Path: \(context.codingPath.map { $0.stringValue })")
    } catch {
        print("❌ Unexpected error: \(error)")
    }
}
```

## Summary

The fix requires updating your Swift data models to:

1. **Use `String` for the `rating` field** (or add a custom decoder to convert it)
2. **Handle `content` as a JSON string** that needs secondary decoding

Apply these changes to your iOS codebase and the plan activation should work correctly.
