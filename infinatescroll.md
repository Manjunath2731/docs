# React Native Mobile App: Infinite Scrolling Integration Guide

## Overview
This guide explains how to implement infinite scrolling for the Organizations API in your React Native mobile app. The API supports offset-based pagination which is perfect for infinite scrolling.

## API Endpoint
```
GET /api/organizations?offset={offset}&limit={limit}&state={state}&city={city}&type={type}
```

## Response Format
```json
{
  "total": 45,
  "totalAllOrganizations": 100,
  "offset": 0,
  "limit": 10,
  "hasMore": true,
  "data": [
    // Array of organization objects
  ]
}
```

## Key Fields for Frontend
- **`offset`**: Current position in the data (starts at 0)
- **`limit`**: Number of items per request (recommended: 10-20)
- **`hasMore`**: Boolean indicating if more data is available
- **`data`**: Array of organizations for current batch

---

## Complete React Native Implementation

```javascript
import React, { useState, useEffect, useCallback } from 'react';
import { 
  FlatList, 
  View, 
  Text, 
  ActivityIndicator, 
  RefreshControl,
  StyleSheet 
} from 'react-native';

const OrganizationsScreen = () => {
  // State Management
  const [organizations, setOrganizations] = useState([]);
  const [currentOffset, setCurrentOffset] = useState(0);
  const [hasMore, setHasMore] = useState(true);
  const [loading, setLoading] = useState(false);
  const [refreshing, setRefreshing] = useState(false);
  const [filters, setFilters] = useState({
    state: '',
    city: '',
    type: ''
  });

  // Load initial data when component mounts
  useEffect(() => {
    loadInitialData();
  }, []);

  // Load initial data (offset = 0)
  const loadInitialData = async () => {
    setLoading(true);
    try {
      const queryParams = buildQueryString(filters);
      const response = await fetch(
        `/api/organizations?offset=0&limit=10${queryParams}`
      );
      const result = await response.json();
      
      setOrganizations(result.data);
      setCurrentOffset(result.limit); // Next offset will be 10
      setHasMore(result.hasMore);
    } catch (error) {
      console.error('Error loading organizations:', error);
    } finally {
      setLoading(false);
    }
  };

  // Load more data when user scrolls to bottom
  const loadMoreData = async () => {
    if (!hasMore || loading) return; // Prevent multiple calls
    
    setLoading(true);
    try {
      const queryParams = buildQueryString(filters);
      const response = await fetch(
        `/api/organizations?offset=${currentOffset}&limit=10${queryParams}`
      );
      const result = await response.json();
      
      // Append new data to existing data
      setOrganizations(prev => [...prev, ...result.data]);
      
      // Update offset for next request
      setCurrentOffset(prev => prev + result.limit);
      
      // Update hasMore status
      setHasMore(result.hasMore);
    } catch (error) {
      console.error('Error loading more organizations:', error);
    } finally {
      setLoading(false);
    }
  };

  // Reset and reload when filters change
  const applyFilters = async (newFilters) => {
    setFilters(newFilters);
    setOrganizations([]);
    setCurrentOffset(0);
    setHasMore(true);
    await loadInitialData();
  };

  // Pull to refresh
  const onRefresh = async () => {
    setRefreshing(true);
    setOrganizations([]);
    setCurrentOffset(0);
    setHasMore(true);
    await loadInitialData();
    setRefreshing(false);
  };

  // Build query string for filters
  const buildQueryString = (filters) => {
    const params = [];
    if (filters.state) params.push(`state=${encodeURIComponent(filters.state)}`);
    if (filters.city) params.push(`city=${encodeURIComponent(filters.city)}`);
    if (filters.type) params.push(`type=${encodeURIComponent(filters.type)}`);
    return params.length > 0 ? `&${params.join('&')}` : '';
  };

  // Render loading footer
  const renderFooter = () => {
    if (!loading) return null;
    return (
      <View style={styles.footer}>
        <ActivityIndicator size="large" color="#007AFF" />
        <Text style={styles.loadingText}>Loading more...</Text>
      </View>
    );
  };

  // Render organization item
  const renderItem = ({ item }) => (
    <View style={styles.itemContainer}>
      <Text style={styles.organizationName}>{item.name}</Text>
      <Text style={styles.organizationAddress}>{item.address}</Text>
      <Text style={styles.branchCount}>
        {item._count?.branches || 0} branches
      </Text>
    </View>
  );

  return (
    <View style={styles.container}>
      <FlatList
        data={organizations}
        renderItem={renderItem}
        keyExtractor={(item) => item.id}
        onEndReached={loadMoreData}
        onEndReachedThreshold={0.1}
        onRefresh={onRefresh}
        refreshing={refreshing}
        ListFooterComponent={renderFooter}
        refreshControl={
          <RefreshControl
            refreshing={refreshing}
            onRefresh={onRefresh}
            colors={['#007AFF']}
          />
        }
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
  },
  itemContainer: {
    backgroundColor: 'white',
    padding: 15,
    marginHorizontal: 10,
    marginVertical: 5,
    borderRadius: 8,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 3,
  },
  organizationName: {
    fontSize: 18,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 5,
  },
  organizationAddress: {
    fontSize: 14,
    color: '#666',
    marginBottom: 5,
  },
  branchCount: {
    fontSize: 12,
    color: '#007AFF',
    fontWeight: '500',
  },
  footer: {
    padding: 20,
    alignItems: 'center',
  },
  loadingText: {
    marginTop: 10,
    color: '#666',
    fontSize: 14,
  },
});

export default OrganizationsScreen;
```

---

## Key Implementation Points

### 1. Offset Management
```javascript
// Initial load: offset = 0
const response = await fetch('/api/organizations?offset=0&limit=10');

// After successful response:
setCurrentOffset(result.limit); // Next offset = 10

// Load more: offset = 10, then 20, then 30...
const response = await fetch(`/api/organizations?offset=${currentOffset}&limit=10`);
```

### 2. Data Accumulation
```javascript
// Always append new data to existing data
setOrganizations(prev => [...prev, ...result.data]);
```

### 3. hasMore Logic
```javascript
// Stop loading when hasMore becomes false
if (!hasMore || loading) return;
```

### 4. Filter Handling
```javascript
// When filters change, reset everything
const applyFilters = async (newFilters) => {
  setFilters(newFilters);
  setOrganizations([]);        // Clear existing data
  setCurrentOffset(0);         // Reset offset
  setHasMore(true);           // Reset hasMore
  await loadInitialData();    // Load fresh data
};
```

---

## API Call Examples

### Initial Load
```
GET /api/organizations?offset=0&limit=10
```

### Load More (after scrolling)
```
GET /api/organizations?offset=10&limit=10
GET /api/organizations?offset=20&limit=10
GET /api/organizations?offset=30&limit=10
```

### With Filters
```
GET /api/organizations?offset=0&limit=10&state=karnataka&city=bangalore
```

---

## Testing Checklist

- [ ] **Initial Load**: Should show first 10 organizations
- [ ] **Scroll Down**: Should load next 10 organizations
- [ ] **Continue Scrolling**: Should keep loading until `hasMore: false`
- [ ] **Pull to Refresh**: Should reset and reload from beginning
- [ ] **Apply Filters**: Should reset pagination and show filtered results
- [ ] **Loading States**: Should show loading indicators
- [ ] **Error Handling**: Should handle network errors gracefully

---

## Important Notes

1. **Authentication**: Make sure to include JWT token in headers
2. **Error Handling**: Always wrap API calls in try-catch blocks
3. **Performance**: Use `onEndReachedThreshold={0.1}` for optimal scroll detection
4. **Memory**: The app accumulates data, so consider implementing data cleanup for very large lists
5. **Network**: Handle offline scenarios and network timeouts

---

## Troubleshooting

### Common Issues:
1. **Duplicate Data**: Make sure to append data correctly with spread operator
2. **Multiple API Calls**: Check `loading` state to prevent simultaneous requests
3. **Filter Reset**: Always reset offset and data when filters change
4. **Memory Leaks**: Clean up any timers or listeners in useEffect cleanup

### Debug Tips:
- Log `currentOffset` and `hasMore` values
- Check network requests in React Native debugger
- Monitor state changes with React DevTools

---

## Contact
If you have any questions or need clarification on any part of this implementation, please reach out to the backend team.
