/* KON Real Estate Hub — Expo App.js (Extended)

This is a single-file Expo React Native app that demonstrates:

Firebase initialization (Auth, Firestore, Storage)

Email/password auth (sign-up / sign-in) and simple user session

Listings CRUD (local + Firestore sync example)

Image picker + upload to Firebase Storage

Map screen using react-native-maps (markers from listings)

Basic real-time chat using Firestore (conversations/messages)


IMPORTANT: Replace firebaseConfig with your Firebase project values. For Google maps on Android in managed Expo, add your API key to app.json per Expo docs.

Install dependencies (run in project root):

npx create-expo-app kon-real-mobile cd kon-real-mobile npx expo install @react-navigation/native @react-navigation/native-stack react-native-screens react-native-safe-area-context @react-native-async-storage/async-storage expo-image-picker expo-constants expo-permissions expo-web-browser react-native-maps npm install firebase axios npx expo start

Then paste this file as App.js (overwrite). This file is intentionally opinionated to keep the prototype simple. */

import React, { useEffect, useState, useRef } from 'react'; import { SafeAreaView, View, Text, TextInput, Button, FlatList, TouchableOpacity, StyleSheet, Image, Alert, ActivityIndicator } from 'react-native'; import AsyncStorage from '@react-native-async-storage/async-storage'; import * as ImagePicker from 'expo-image-picker'; import * as WebBrowser from 'expo-web-browser'; import Constants from 'expo-constants'; import MapView, { Marker } from 'react-native-maps';

import { NavigationContainer } from '@react-navigation/native'; import { createNativeStackNavigator } from '@react-navigation/native-stack';

// Firebase v9 modular import { initializeApp } from 'firebase/app'; import { getAuth, createUserWithEmailAndPassword, signInWithEmailAndPassword, signOut, onAuthStateChanged } from 'firebase/auth'; import { getFirestore, collection, addDoc, getDocs, doc, setDoc, query, where, onSnapshot, orderBy, serverTimestamp } from 'firebase/firestore'; import { getStorage, ref, uploadBytes, getDownloadURL } from 'firebase/storage';

// ----------------- CONFIG ----------------- const firebaseConfig = { apiKey: "REPLACE_WITH_YOUR_API_KEY", authDomain: "REPLACE_WITH_YOUR_AUTH_DOMAIN", projectId: "REPLACE_WITH_YOUR_PROJECT_ID", storageBucket: "REPLACE_WITH_YOUR_STORAGE_BUCKET", messagingSenderId: "REPLACE_WITH_YOUR_MESSAGING_SENDER_ID", appId: "REPLACE_WITH_YOUR_APP_ID" };

const app = initializeApp(firebaseConfig); const auth = getAuth(app); const db = getFirestore(app); const storage = getStorage(app);

// Sample fallback listings (used if Firestore is empty or for first-run) const SAMPLE_LISTINGS = [ { id: 'p1', title: '3-Bedroom Apartment', type: 'Apartment', price: 250000, currency: 'NGN', location: 'Rukuba Barracks, Jos', status: 'For Sale', description: 'Spacious 3-bedroom apartment near market and schools.', bedrooms: 3, bathrooms: 2, size: '120 sqm', lat: 9.9, lng: 8.9, images: [] }, { id: 'p2', title: 'Vacant Land (500 sqm)', type: 'Land', price: 80000, currency: 'NGN', location: 'Agingi Market (Rukuba)', status: 'For Sale', description: 'Prime land, great for residential development.', bedrooms: 0, bathrooms: 0, size: '500 sqm', lat: 9.88, lng: 8.85, images: [] }, ];

// ----------------- Helpers ----------------- async function uploadImageAsync(uri, path = 'images') { // fetch blob const response = await fetch(uri); const blob = await response.blob(); const filename = ${path}/${Date.now()}-${Math.random().toString(36).slice(2, 9)}.jpg; const storageRef = ref(storage, filename); await uploadBytes(storageRef, blob); const url = await getDownloadURL(storageRef); return url; }

// ----------------- Screens ----------------- function LoadingScreen() { return ( <SafeAreaView style={styles.centered}><ActivityIndicator size="large" /></SafeAreaView> ); }

function AuthScreen({ navigation }) { const [isSignUp, setIsSignUp] = useState(false); const [email, setEmail] = useState(''); const [password, setPassword] = useState(''); const [loading, setLoading] = useState(false);

async function handleAuth() { setLoading(true); try { if (isSignUp) { const userCred = await createUserWithEmailAndPassword(auth, email, password); console.log('signed up', userCred.user.uid); } else { const userCred = await signInWithEmailAndPassword(auth, email, password); console.log('signed in', userCred.user.uid); } } catch (e) { Alert.alert('Auth error', e.message); } finally { setLoading(false); } }

return ( <SafeAreaView style={styles.container}> <Text style={styles.title}>{isSignUp ? 'Create account' : 'Sign in'}</Text> <TextInput placeholder="Email" value={email} onChangeText={setEmail} style={styles.input} autoCapitalize='none' /> <TextInput placeholder="Password" value={password} onChangeText={setPassword} style={styles.input} secureTextEntry /> <Button title={isSignUp ? 'Sign up' : 'Sign in'} onPress={handleAuth} disabled={loading} /> <View style={{ height: 12 }} /> <Button title={isSignUp ? 'Have an account? Sign in' : "Don't have an account? Sign up"} onPress={() => setIsSignUp(s => !s)} /> </SafeAreaView> ); }

function HomeScreen({ navigation, user }) { const [listings, setListings] = useState([]); const [queryText, setQueryText] = useState(''); const [loading, setLoading] = useState(true);

useEffect(() => { // subscribe to Firestore listings collection const q = query(collection(db, 'listings'), orderBy('createdAt', 'desc')); const unsub = onSnapshot(q, (snap) => { if (snap.empty) { // seed sample listings locally if none setListings(SAMPLE_LISTINGS); setLoading(false); return; } const arr = snap.docs.map(d => ({ id: d.id, ...d.data() })); setListings(arr); setLoading(false); }, (err) => { console.error('listings err', err); setLoading(false); });

return () => unsub();

}, []);

const filtered = listings.filter(l => (l.title + ' ' + l.location + ' ' + l.description).toLowerCase().includes(queryText.toLowerCase()));

return ( <SafeAreaView style={styles.container}> <View style={styles.row}> <TextInput placeholder="Search listings" value={queryText} onChangeText={setQueryText} style={styles.input} /> <Button title="Map" onPress={() => navigation.navigate('Map', { listings })} /> </View>

<View style={{ flexDirection: 'row', gap: 8, marginBottom: 8 }}>
    <Button title="Add" onPress={() => navigation.navigate('AddListing')} />
    <View style={{ width: 8 }} />
    <Button title="Agent" onPress={() => navigation.navigate('AgentDashboard')} />
    <View style={{ width: 8 }} />
    <Button title="Messages" onPress={() => navigation.navigate('Conversations')} />
  </View>

  {loading ? <ActivityIndicator /> : (
    <FlatList
      data={filtered}
      keyExtractor={i => i.id}
      renderItem={({ item }) => (
        <TouchableOpacity style={styles.card} onPress={() => navigation.navigate('PropertyDetails', { id: item.id })}>
          {item.images && item.images[0] ? <Image source={{ uri: item.images[0] }} style={styles.cardImage} /> : <View style={[styles.cardImage, { backgroundColor: '#eee', alignItems: 'center', justifyContent: 'center' }]}><Text>No image</Text></View>}
          <View style={{ flex: 1, paddingLeft: 8 }}>
            <Text style={styles.cardTitle}>{item.title}</Text>
            <Text style={styles.cardSub}>{item.location} • {item.size || ''}</Text>
            <Text style={styles.cardPrice}>{item.currency} {item.price?.toLocaleString?.() ?? item.price}</Text>
          </View>
        </TouchableOpacity>
      )}
    />
  )}
</SafeAreaView>

); }

function PropertyDetailsScreen({ route, navigation }) { const { id } = route.params; const [listing, setListing] = useState(null); const [loading, setLoading] = useState(true);

useEffect(() => { const d = doc(db, 'listings', id); const unsub = onSnapshot(d, (snap) => { if (!snap.exists()) { setListing(null); setLoading(false); return; } setListing({ id: snap.id, ...snap.data() }); setLoading(false); }); return () => unsub(); }, [id]);

if (loading) return <SafeAreaView style={styles.centered}><ActivityIndicator /></SafeAreaView>; if (!listing) return <SafeAreaView style={styles.centered}><Text>Listing not found</Text></SafeAreaView>;

return ( <SafeAreaView style={styles.container}> <FlatList data={listing.images && listing.images.length ? listing.images : []} horizontal renderItem={({ item }) => <Image source={{ uri: item }} style={{ width: 300, height: 200, marginRight: 8, borderRadius: 8 }} />} keyExtractor={(i, idx) => ${idx}} /> <Text style={styles.title}>{listing.title}</Text> <Text style={styles.sub}>{listing.location}</Text> <Text style={{ marginTop: 8 }}>{listing.description}</Text> <View style={{ marginTop: 12 }}> <Button title="Message Agent" onPress={() => navigation.navigate('Chat', { conversationId: conv_${listing.id}, listing })} /> </View> </SafeAreaView> ); }

function AddListingScreen({ navigation }) { const [title, setTitle] = useState(''); const [price, setPrice] = useState(''); const [location, setLocation] = useState(''); const [lat, setLat] = useState(null); const [lng, setLng] = useState(null); const [images, setImages] = useState([]); const [uploading, setUploading] = useState(false);

async function pickImage() { const perm = await ImagePicker.requestMediaLibraryPermissionsAsync(); if (perm.status !== 'granted') return Alert.alert('Permission required'); const res = await ImagePicker.launchImageLibraryAsync({ quality: 0.7, allowsEditing: true }); if (res.cancelled) return; setImages(i => [res.uri, ...i]); }

async function handlePost() { if (!title || !price) return Alert.alert('Provide title and price'); setUploading(true); try { // upload images const uploadedUrls = []; for (let uri of images) { const url = await uploadImageAsync(uri); uploadedUrls.push(url); } const docRef = await addDoc(collection(db, 'listings'), { title, price: Number(price), currency: 'NGN', location, lat, lng, images: uploadedUrls, description: '', createdAt: serverTimestamp(), status: 'For Sale' }); Alert.alert('Posted', 'Listing created'); navigation.navigate('PropertyDetails', { id: docRef.id }); } catch (e) { console.error(e); Alert.alert('Upload error', e.message); } finally { setUploading(false); } }

return ( <SafeAreaView style={styles.container}> <TextInput placeholder="Title" value={title} onChangeText={setTitle} style={styles.input} /> <TextInput placeholder="Price" value={price} onChangeText={setPrice} style={styles.input} keyboardType='numeric' /> <TextInput placeholder="Location (address)" value={location} onChangeText={setLocation} style={styles.input} /> <View style={{ flexDirection: 'row', gap: 8 }}> <Button title="Pick Image" onPress={pickImage} /> <Button title="Set on Map" onPress={() => navigation.navigate('PickLocation', { onPicked: ({ latitude, longitude }) => { setLat(latitude); setLng(longitude); } })} /> </View> <View style={{ height: 12 }} /> <FlatList data={images} horizontal keyExtractor={(i, idx) => ${idx}} renderItem={({ item }) => <Image source={{ uri: item }} style={{ width: 120, height: 80, marginRight: 8 }} />} /> <View style={{ height: 12 }} /> <Button title={uploading ? 'Uploading...' : 'Post Listing'} onPress={handlePost} disabled={uploading} /> </SafeAreaView> ); }

function PickLocationScreen({ route, navigation }) { const onPicked = route.params?.onPicked; const [region, setRegion] = useState({ latitude: 9.8965, longitude: 8.8583, latitudeDelta: 0.05, longitudeDelta: 0.05 });

return ( <SafeAreaView style={styles.container}> <MapView style={{ flex: 1 }} initialRegion={region} onRegionChangeComplete={setRegion}> <Marker coordinate={{ latitude: region.latitude, longitude: region.longitude }} draggable onDragEnd={(e) => setRegion({ ...region, latitude: e.nativeEvent.coordinate.latitude, longitude: e.nativeEvent.coordinate.longitude })} /> </MapView> <Button title="Pick this location" onPress={() => { onPicked({ latitude: region.latitude, longitude: region.longitude }); navigation.goBack(); }} /> </SafeAreaView> ); }

function MapScreen({ route }) { const listings = route.params?.listings || []; const initial = { latitude: 9.8965, longitude: 8.8583, latitudeDelta: 0.5, longitudeDelta: 0.5 };

return ( <SafeAreaView style={{ flex: 1 }}> <MapView style={{ flex: 1 }} initialRegion={initial}> {listings.map(l => (l.lat && l.lng ? <Marker key={l.id} coordinate={{ latitude: l.lat, longitude: l.lng }} title={l.title} description={l.location} /> : null))} </MapView> </SafeAreaView> ); }

function AgentDashboardScreen() { const [listings, setListings] = useState([]);

useEffect(() => { // query listings where agentId == current user id (for prototype we show all) const q = query(collection(db, 'listings'), orderBy('createdAt', 'desc')); const unsub = onSnapshot(q, snap => setListings(snap.docs.map(d => ({ id: d.id, ...d.data() })))); return () => unsub(); }, []);

return ( <SafeAreaView style={styles.container}> <Text style={styles.title}>Agent Dashboard</Text> <FlatList data={listings} keyExtractor={i => i.id} renderItem={({ item }) => ( <View style={styles.card}><Text style={styles.cardTitle}>{item.title}</Text><Text>{item.location}</Text></View> )} /> </SafeAreaView> ); }

// ----------------- Chat ----------------- function ConversationsScreen({ navigation }) { const [conversations, setConversations] = useState([]);

useEffect(() => { const q = query(collection(db, 'conversations'), orderBy('lastUpdated', 'desc')); const unsub = onSnapshot(q, snap => setConversations(snap.docs.map(d => ({ id: d.id, ...d.data() })))); return () => unsub(); }, []);

return ( <SafeAreaView style={styles.container}> <FlatList data={conversations} keyExtractor={i => i.id} renderItem={({ item }) => ( <TouchableOpacity style={styles.card} onPress={() => navigation.navigate('Chat', { conversationId: item.id })}> <Text style={styles.cardTitle}>{item.title || 'Conversation'}</Text> <Text>{item.participants?.join(', ')}</Text> </TouchableOpacity> )} /> </SafeAreaView> ); }

function ChatScreen({ route }) { const conversationId = route.params?.conversationId; const listing = route.params?.listing; const [messages, setMessages] = useState([]); const [text, setText] = useState('');

useEffect(() => { if (!conversationId) return; const q = query(collection(db, conversations/${conversationId}/messages), orderBy('createdAt', 'asc')); const unsub = onSnapshot(q, snap => setMessages(snap.docs.map(d => ({ id: d.id, ...d.data() })))); return () => unsub(); }, [conversationId]);

async function send() { if (!text.trim()) return; await addDoc(collection(db, conversations/${conversationId}/messages), { text, senderId: 'anonymous', createdAt: serverTimestamp() }); await setDoc(doc(db, 'conversations', conversationId), { title: listing?.title || 'Conversation', lastUpdated: serverTimestamp() }, { merge: true }); setText(''); }

return ( <SafeAreaView style={styles.container}> <FlatList data={messages} keyExtractor={i => i.id} renderItem={({ item }) => <View style={{ padding: 8 }}><Text style={{ fontWeight: '700' }}>{item.senderId}</Text><Text>{item.text}</Text></View>} /> <View style={{ flexDirection: 'row', gap: 8, alignItems: 'center' }}> <TextInput placeholder='Message' value={text} onChangeText={setText} style={[styles.input, { flex: 1 }]} /> <Button title='Send' onPress={send} /> </View> </SafeAreaView> ); }

// ----------------- App ----------------- const Stack = createNativeStackNavigator();

export default function App() { const [user, setUser] = useState(null); const [initializing, setInitializing] = useState(true);

useEffect(() => { const unsub = onAuthStateChanged(auth, (u) => { setUser(u); if (initializing) setInitializing(false); }); return () => unsub(); }, []);

if (initializing) return <LoadingScreen />;

return ( <NavigationContainer> <Stack.Navigator> {!user ? ( <Stack.Screen name="Auth" component={AuthScreen} options={{ headerShown: false }} /> ) : ( <> <Stack.Screen name="Home" options={{ title: 'KON Real Estate' }}> {(props) => <HomeScreen {...props} user={user} />} </Stack.Screen> <Stack.Screen name="PropertyDetails" component={PropertyDetailsScreen} options={{ title: 'Property' }} /> <Stack.Screen name="AddListing" component={AddListingScreen} options={{ title: 'Add Listing' }} /> <Stack.Screen name="PickLocation" component={PickLocationScreen} options={{ title: 'Pick Location' }} /> <Stack.Screen name="Map" component={MapScreen} options={{ title: 'Map' }} /> <Stack.Screen name="AgentDashboard" component={AgentDashboardScreen} options={{ title: 'Agent' }} /> <Stack.Screen name="Conversations" component={ConversationsScreen} options={{ title: 'Conversations' }} /> <Stack.Screen name="Chat" component={ChatScreen} options={{ title: 'Chat' }} /> <Stack.Screen name="PropertyDetails" component={PropertyDetailsScreen} /> </> )} </Stack.Navigator> </NavigationContainer> ); }

// ----------------- Styles ----------------- const styles = StyleSheet.create({ container: { flex: 1, padding: 12, backgroundColor: '#fff' }, centered: { flex: 1, alignItems: 'center', justifyContent: 'center' }, row: { flexDirection: 'row', alignItems: 'center', marginBottom: 8 }, input: { borderWidth: 1, borderColor: '#ddd', padding: 8, borderRadius: 6, marginBottom: 8 }, card: { flexDirection: 'row', padding: 8, borderRadius: 8, borderWidth: 1, borderColor: '#eee', marginBottom: 8, alignItems: 'center' }, cardImage: { width: 96, height: 72, borderRadius: 6 }, cardTitle: { fontWeight: '700', fontSize: 16 }, cardSub: { color: '#666' }, cardPrice: { marginTop: 8, fontWeight: '700' }, title: { fontSize: 20, fontWeight: '700' }, sub: { color: '#666' }, });

