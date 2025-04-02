import React, { useState, useEffect, useRef } from 'react';
import firebase from 'firebase/app';
import 'firebase/firestore';

// Initialize Firebase – replace with your own config!
if (!firebase.apps.length) {
  firebase.initializeApp({
    apiKey: "YOUR_API_KEY",
    authDomain: "YOUR_AUTH_DOMAIN",
    projectId: "YOUR_PROJECT_ID",
    // ...other config values
  });
}
const firestore = firebase.firestore();

const configuration = {
  iceServers: [{ urls: 'stun:stun.l.google.com:19302' }],
};

const App = () => {
  const [stream, setStream] = useState(null);
  const [isLawyer, setIsLawyer] = useState(false);
  const [incomingCall, setIncomingCall] = useState(false);
  const [isAvailable, setIsAvailable] = useState(false);
  const localVideoRef = useRef(null);
  const pc = useRef(new RTCPeerConnection(configuration));

  // Firestore documents for signaling
  const callDoc = firestore.collection('calls').doc('activeCall');
  const lawyerStatusDoc = firestore.collection('lawyers').doc('status');

  useEffect(() => {
    requestNotificationPermission();
    // Listen for call document updates as a substitute for push notifications
    const unsubscribe = callDoc.onSnapshot((doc) => {
      if (doc.exists) {
        const data = doc.data();
        // If there is an incoming call flag set and the lawyer is available, show UI notification.
        if (data.incomingCall && isAvailable) {
          setIncomingCall(true);
          showBrowserNotification('Incoming Call', 'A client is calling you.');
        }
      }
    });
    return () => unsubscribe();
  }, [isAvailable]);

  const requestNotificationPermission = async () => {
    if ('Notification' in window) {
      const permission = await Notification.requestPermission();
      if (permission === 'granted') {
        console.log('Browser notification permission granted');
      }
    }
  };

  const showBrowserNotification = (title, body) => {
    if ('Notification' in window && Notification.permission === 'granted') {
      new Notification(title, { body });
    }
  };

  const toggleAvailability = async () => {
    const newAvailability = !isAvailable;
    setIsAvailable(newAvailability);
    await lawyerStatusDoc.set({ available: newAvailability });
  };

  const startVideoCall = async () => {
    try {
      // Check for available lawyers
      const availableLawyers = await firestore
        .collection('lawyers')
        .where('available', '==', true)
        .get();
      if (availableLawyers.empty) {
        alert('No available lawyers at the moment. Please try again later.');
        return;
      }
      // Get local media stream
      const localStream = await navigator.mediaDevices.getUserMedia({
        video: true,
        audio: true,
      });
      setStream(localStream);
      if (localVideoRef.current) {
        localVideoRef.current.srcObject = localStream;
      }
      localStream.getTracks().forEach((track) =>
        pc.current.addTrack(track, localStream)
      );
      // Create offer
      const offer = await pc.current.createOffer();
      await pc.current.setLocalDescription(offer);
      // Save offer and flag incoming call in Firestore
      await callDoc.set({ offer, incomingCall: true });
    } catch (error) {
      console.error('Error starting video call:', error);
    }
  };

  const answerCall = async () => {
    try {
      setIsLawyer(true);
      setIncomingCall(false);
      // Get local media stream
      const localStream = await navigator.mediaDevices.getUserMedia({
        video: true,
        audio: true,
      });
      setStream(localStream);
      if (localVideoRef.current) {
        localVideoRef.current.srcObject = localStream;
      }
      localStream.getTracks().forEach((track) =>
        pc.current.addTrack(track, localStream)
      );
      const callData = await callDoc.get();
      if (!callData.exists) return;
      // Set remote description from the offer
      await pc.current.setRemoteDescription(
        new RTCSessionDescription(callData.data().offer)
      );
      // Create and send answer
      const answer = await pc.current.createAnswer();
      await pc.current.setLocalDescription(answer);
      await callDoc.update({ answer, incomingCall: false });
    } catch (error) {
      console.error('Error answering call:', error);
    }
  };

  const declineCall = () => {
    setIncomingCall(false);
    callDoc.update({ incomingCall: false });
  };

  return (
    <div style={{ textAlign: 'center', padding: '20px' }}>
      <h1>Lawyer On Demand</h1>
      {isLawyer ? (
        <div>
          <button onClick={toggleAvailability}>
            {isAvailable ? 'Go Offline' : 'Go Online'}
          </button>
          {incomingCall ? (
            <div
              style={{
                border: '1px solid #ccc',
                padding: '20px',
                margin: '20px auto',
                width: '300px',
              }}
            >
              <p>Incoming Call</p>
              <button onClick={answerCall} style={{ marginRight: '10px' }}>
                Accept
              </button>
              <button onClick={declineCall}>Decline</button>
            </div>
          ) : (
            <p>No Incoming Calls</p>
          )}
        </div>
      ) : (
        <button onClick={startVideoCall}>Call Lawyer</button>
      )}
      {stream && (
        <video
          ref={localVideoRef}
          autoPlay
          playsInline
          style={{
            width: '300px',
            height: '400px',
            backgroundColor: 'black',
            marginTop: '20px',
          }}
        ></video>
      )}
    </div>
  );
};

export default App;
