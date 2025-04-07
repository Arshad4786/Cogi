'use client';

import React, { useState, useEffect, useCallback } from 'react';
import Sidebar from '../../components/Sidebar';
import FileBar from '../../components/FileBar';
import Topbar from '../../components/Topbar';
import ChatSection from '../../components/ChatSection';
import Bottombar from '../../components/Bottombar';
import HomeTabContent from '../../components/HomeTabContent';
import CodeDisplay from '../../components/CodeDisplay';
import { FiLoader } from 'react-icons/fi'; // Keep for loading indicator

// Placeholder view remains the same
const PlaceholderView = ({ tabName }) => (
  <div className="flex flex-1 flex-col items-center justify-center p-6 text-gray-500 dark:text-gray-400 bg-gray-50 dark:bg-gray-900 overflow-y-auto">
    <h2 className="text-2xl font-semibold mb-2">Welcome to {tabName}</h2>
    <p>Content for {tabName} will be displayed here.</p>
  </div>
);

export default function Home() {
  // --- Authentication State (Managed Manually) ---
  const [accessToken, setAccessToken] = useState(null); // Stores the GitHub token
  const [sessionStatus, setSessionStatus] = useState('loading'); // 'loading', 'authenticated', 'unauthenticated'
  const [userInfo, setUserInfo] = useState(null); // Stores { name: string|null, image: string|null }

  // --- UI and Data State ---
  const [activeTab, setActiveTab] = useState('Home');
  const [selectedRepo, setSelectedRepo] = useState(null); // Store the full repo object or null
  const [userRepos, setUserRepos] = useState([]);
  const [isRepoListLoading, setIsRepoListLoading] = useState(false);
  const [repoListError, setRepoListError] = useState(null);

  const [selectedFile, setSelectedFile] = useState(null);
  const [fileContent, setFileContent] = useState(null);
  const [isFileLoading, setIsFileLoading] = useState(false);
  const [fileError, setFileError] = useState(null);

  // --- Electron API Interaction & Auth Handling ---

  // Function to trigger login via Electron
  const handleLogin = useCallback(() => {
    if (window.electronAPI && typeof window.electronAPI.loginGithub === 'function') {
      console.log("Page: Requesting GitHub login via Electron...");
      setSessionStatus('loading'); // Indicate authentication is in progress
      setAccessToken(null); // Clear any old token
      setUserInfo(null); // Clear old user info
      // Clear repo/file state as well during login attempt
      setUserRepos([]);
      setSelectedRepo(null);
      setRepoListError(null);
      handleClearFile();
      window.electronAPI.loginGithub();
    } else {
      console.error('Electron API (window.electronAPI or loginGithub) not found.');
      // Set status to unauthenticated if API isn't available to show login prompt
      setSessionStatus('unauthenticated');
      setRepoListError("Could not initiate login: Electron API unavailable."); // Show error
    }
  }, []); // No dependencies, this function is stable

  // Effect to listen for the token from Electron main process
  useEffect(() => {
    let unsubscribe;
    let cleanupPerformed = false; // Flag to prevent state updates after unmount

    // Define the listener function separately
    const handleToken = (token) => {
      if (cleanupPerformed) return; // Don't update state if unmounted

      console.log('Page: Token received from Electron:', token ? 'Yes' : 'No');
      if (token) {
        setAccessToken(token);
        setSessionStatus('authenticated'); // Token received, user is authenticated
        setRepoListError(null); // Clear any previous errors like "API unavailable"
      } else {
        // This could happen if login fails, or if the app starts without a stored token
        setAccessToken(null);
        setUserInfo(null); // Clear user info if token is removed/null
        setSessionStatus('unauthenticated'); // No token means unauthenticated
        // Keep existing repo/file state? Or clear? Clearing might be safer.
         setUserRepos([]);
         setSelectedRepo(null);
         setRepoListError(null); // Clear errors if we explicitly get a null token
         handleClearFile();
      }
    };

    // Check for API and set up listener
    if (window.electronAPI && typeof window.electronAPI.onGithubToken === 'function') {
      console.log('Page: Setting up GitHub token listener...');
      unsubscribe = window.electronAPI.onGithubToken(handleToken);

      // Optional: Request current token state immediately on load?
      if (typeof window.electronAPI.getCurrentToken === 'function') {
        window.electronAPI.getCurrentToken().then(handleToken).catch(err => {
          console.error("Error requesting current token:", err);
          if (!cleanupPerformed) setSessionStatus('unauthenticated'); // Assume unauth if request fails
        });
      } else {
         // If we can't request current token, assume unauthenticated until listener fires
         setSessionStatus('unauthenticated');
      }
      // Simplified: Assume loading until the listener gets a chance to run or fetch happens
      // The fetchUserRepos effect will handle initial state based on token presence

    } else {
      console.warn('Page: Electron API or onGithubToken function not found during effect setup.');
      // If the API isn't there, we can't authenticate via Electron
       setSessionStatus('unauthenticated');
       setRepoListError("Authentication unavailable: Required Electron features missing.");
    }

    // Cleanup function
    return () => {
      cleanupPerformed = true;
      if (unsubscribe && typeof unsubscribe === 'function') {
        console.log('Page: Cleaning up GitHub token listener...');
        unsubscribe();
      }
    };
  }, []); // Run only on mount

  // --- Fetch User Info (Name, Avatar) ---
  useEffect(() => {
    const fetchUserInfo = async () => {
      if (!accessToken) {
        setUserInfo(null); // Clear user info if no token
        return;
      }

      console.log("Page: Fetching user info...");
      try {
        const response = await fetch('https://api.github.com/user', {
          headers: {
            Authorization: `Bearer ${accessToken}`,
            Accept: 'application/vnd.github.v3+json',
          },
        });
        if (!response.ok) {
          // Handle token errors (e.g., expired, revoked)
          if (response.status === 401 || response.status === 403) {
            console.error(`Page: Invalid token (${response.status}). Clearing token and setting unauthenticated.`);
            setAccessToken(null); // Clear the invalid token
            setSessionStatus('unauthenticated');
            setUserInfo(null);
             // Clear other sensitive data
             setUserRepos([]);
             setSelectedRepo(null);
             setRepoListError("Authentication failed. Please log in again.");
             handleClearFile();
          } else {
             throw new Error(`Failed to fetch user info (Status: ${response.status})`);
          }
          return; // Stop processing if response not ok
        }
        const data = await response.json();
        setUserInfo({
          name: data.name || data.login, // Use name, fallback to login
          image: data.avatar_url,
        });
        console.log("Page: User info fetched:", data.login);
      } catch (err) {
        console.error('Page: Error fetching user info:', err);
        setUserInfo(null); // Clear user info on error
        // Optionally set a general error state or specific user info error
      }
    };

    fetchUserInfo();
  }, [accessToken]); // Rerun when accessToken changes

  // --- Fetch User Repositories ---
  useEffect(() => {
    const fetchUserRepos = async () => {
      // Only fetch if we have a token and are considered authenticated
      if (accessToken && sessionStatus === 'authenticated') {
        console.log("Page: Fetching user repos with token...");
        setIsRepoListLoading(true);
        setRepoListError(null);
        setUserRepos([]); // Clear previous repos
        // Don't clear selectedRepo here initially, maybe user selected one before token refresh?
        // Clear file selection as repo list is refreshing
        handleClearFile();

        try {
          const response = await fetch('https://api.github.com/user/repos?sort=updated&per_page=100', {
            headers: {
              Authorization: `Bearer ${accessToken}`, // Use the state token
              'Accept': 'application/vnd.github.v3+json',
            },
          });

          if (!response.ok) {
             // Handle specific errors like invalid token during repo fetch
             if (response.status === 401 || response.status === 403) {
               console.error(`Page: Invalid token during repo fetch (${response.status}). Clearing token.`);
               setAccessToken(null); // Clear invalid token
               setSessionStatus('unauthenticated');
               setUserInfo(null);
               setUserRepos([]);
               setSelectedRepo(null);
               setRepoListError("Authentication failed while fetching repositories. Please log in again.");
               handleClearFile();
               return; // Stop processing
             }
             // Generic error
             let errorMsg = `Failed to fetch repositories (Status: ${response.status})`;
             try { const errorData = await response.json(); errorMsg = errorData.message || errorMsg; } catch (_) {}
             throw new Error(errorMsg);
          }

          const data = await response.json();
          console.log("Page: Repos fetched:", data.length);
          setUserRepos(data);

          // Logic to set default selected repo OR keep existing selection if still valid
          if (selectedRepo && !data.some(repo => repo.id === selectedRepo.id)) {
             // If previously selected repo is NOT in the new list, clear selection
             console.log("Page: Previously selected repo not found in new list. Clearing selection.");
             setSelectedRepo(null);
           } else if (!selectedRepo && data.length > 0) {
             // If no repo was selected AND the new list is not empty, select the first one
             console.log("Page: No repo selected, defaulting to first fetched repo:", data[0]?.full_name);
             setSelectedRepo(data[0]);
           } else if (selectedRepo) {
             // If a repo was selected and it IS in the new list, keep it selected (do nothing)
             console.log("Page: Keeping currently selected repo:", selectedRepo.full_name);
           } else {
              // No repo was selected, and the new list is empty
              console.log("Page: No repositories found for user.");
              setSelectedRepo(null); // Ensure it's null
           }

        } catch (err) {
          // Avoid setting state if the error was due to token invalidation (already handled)
          if (sessionStatus === 'authenticated') {
             console.error('Page: Error fetching repos:', err);
             setRepoListError(err.message);
             setUserRepos([]); // Clear repos on error
             setSelectedRepo(null); // Clear selection on error
          }
        } finally {
          // Avoid setting loading false if status changed due to token error
          if (sessionStatus === 'authenticated') {
             setIsRepoListLoading(false);
             console.log("Page: Finished fetching repos.");
          }
        }
      } else if (!accessToken && sessionStatus !== 'loading') {
          // Clear state if token is explicitly removed or status becomes unauthenticated
          console.log("Page: No access token or not authenticated, clearing repo state.");
          setUserRepos([]);
          setSelectedRepo(null);
          setRepoListError(null);
          setIsRepoListLoading(false);
          handleClearFile();
      }
    };

    fetchUserRepos();
  // Rerun when accessToken changes or status becomes authenticated
  }, [accessToken, sessionStatus]);

  // --- File Handling Logic (Unchanged) ---
  const handleFileSelect = useCallback((file) => {
    console.log("Page: File selected:", file?.path);
    if (file?.path && file?.name && file?.sha) {
      setSelectedFile(file);
      setFileContent(null);
      setFileError(null);
      setIsFileLoading(true); // Start loading file content
    } else {
      console.warn("Page: Invalid file object received for selection:", file);
      handleClearFile();
    }
  }, []); // No dependencies needed

  const handleClearFile = useCallback(() => {
    // Check if already cleared to avoid redundant logs/renders
    if (selectedFile || fileContent || fileError || isFileLoading) {
      console.log("Page: Clearing selected file state.");
      setSelectedFile(null);
      setFileContent(null);
      setFileError(null);
      setIsFileLoading(false);
    }
  }, [selectedFile, fileContent, fileError, isFileLoading]); // Dependencies ensure it has latest state

  const handleFileContentLoaded = useCallback((content, error) => {
    console.log(`Page: File content loaded. Error: ${error ? error : 'None'}`);
    setIsFileLoading(false);
    if (error) {
      setFileError(error);
      setFileContent(null);
    } else {
      setFileContent(content);
      setFileError(null);
    }
  }, []); // No dependencies needed

  // --- Repo Change Effect (Unchanged) ---
  useEffect(() => {
    if (selectedRepo) {
      console.log("Page: Selected repo changed to:", selectedRepo.full_name);
    } else {
      console.log("Page: Selected repo cleared.");
    }
    // Clear file details when repo changes
    handleClearFile();
  }, [selectedRepo, handleClearFile]); // Add handleClearFile dependency

  // --- Tab Change Effect (Unchanged) ---
  useEffect(() => {
    // If a file is selected but the user navigates away from Code/Chat
    if (selectedFile && activeTab !== 'Code' && activeTab !== 'Chat') {
      console.log("Page: Clearing selected file due to tab change away from Code/Chat:", activeTab);
      handleClearFile();
    }
  }, [activeTab, selectedFile, handleClearFile]); // Add handleClearFile dependency


  // --- Render Logic ---
  const renderTabContent = () => {
    // Show loading indicator if session status is loading OR if authenticated but still loading initial repos
    if (sessionStatus === 'loading' || (sessionStatus === 'authenticated' && isRepoListLoading && userRepos.length === 0)) {
      return (
        <div className="flex flex-1 items-center justify-center bg-gray-50 dark:bg-gray-900">
            <FiLoader size={32} className="animate-spin text-blue-500" />
            <span className="ml-3 text-gray-600 dark:text-gray-400">
                {sessionStatus === 'loading' ? 'Authenticating...' : 'Loading repositories...'}
            </span>
        </div>
      );
    }

     // If unauthenticated after loading, show HomeTabContent which will render the login prompt
     if (sessionStatus === 'unauthenticated' && activeTab !== 'Home') {
         // Force redirect to Home tab if trying to access other tabs while logged out
         // Or just show a generic login required message if preferred
         console.log("Page: Unauthenticated, switching to Home tab.");
         setActiveTab('Home'); // Switch tab automatically
         // Alternatively, show a message:
         // return (
         //   <div className="flex flex-1 items-center justify-center p-6 text-gray-500 dark:text-gray-400 bg-gray-50 dark:bg-gray-900">
         //     Please log in to access this section.
         //   </div>
         // );
         // Returning null here will cause a quick re-render as setActiveTab updates state
         return null;
     }


    switch (activeTab) {
      case 'Home':
        return (
          <HomeTabContent
            // Pass state and setters related to repo list and selection
            userRepos={userRepos}
            selectedRepo={selectedRepo}
            setSelectedRepo={setSelectedRepo}
            isRepoListLoading={isRepoListLoading}
            repoListError={repoListError}
            // Pass the manually managed session status and the login handler
            sessionStatus={sessionStatus}
            accessToken={accessToken} // Pass token if Home needs it (e.g., for README fetch)
            onLoginClick={handleLogin} // Pass the login function
          />
        );
      case 'Chat':
      case 'Code':
        // Check authentication status before rendering these tabs
        if (sessionStatus !== 'authenticated') {
          // This case is unlikely if the check above switches to Home, but as a fallback:
           return (
             <div className="flex flex-1 items-center justify-center p-6 text-gray-500 dark:text-gray-400 bg-gray-50 dark:bg-gray-900">
               Please log in to view code or chat.
             </div>
           );
        }
        if (!selectedRepo) {
          return (
            <div className="flex flex-1 items-center justify-center p-6 text-gray-500 dark:text-gray-400 bg-gray-50 dark:bg-gray-900">
              Please select a repository first (from Topbar or Home tab).
            </div>
          );
        }
        return (
          <div className="flex flex-1 overflow-hidden">
            <FileBar
              selectedRepo={selectedRepo?.full_name}
              onFileSelect={handleFileSelect}
              selectedFile={selectedFile}
              onFileContentLoaded={handleFileContentLoaded}
              // tabName={activeTab} // Not strictly needed by FileBar
              accessToken={accessToken} // Pass token for API calls within FileBar
            />
            {activeTab === 'Chat' ? (
              <ChatSection
                selectedRepoFullName={selectedRepo?.full_name}
                accessToken={accessToken} // Pass token for API calls within ChatSection
                selectedFile={selectedFile}
                selectedFileContent={fileContent}
                isLoading={isFileLoading}
                error={fileError}
              />
            ) : ( // activeTab === 'Code'
              <CodeDisplay
                selectedFile={selectedFile}
                fileContent={fileContent}
                onClearFile={handleClearFile} // Pass clear function
                isLoading={isFileLoading}
                error={fileError}
              />
            )}
          </div>
        );
      // Other placeholder tabs
      case 'Notifications':
      case 'Settings':
      case 'Automation':
        return <PlaceholderView tabName={activeTab} />;
      default:
        console.warn("Page: Rendering unknown tab:", activeTab);
        // Fallback to Home tab if state gets corrupted somehow
        setActiveTab('Home');
        return null; // Will re-render with Home
    }
  };

  // --- Component Return ---
  return (
    <div className="flex flex-col h-screen bg-white dark:bg-gray-900 overflow-hidden">
      <div className="flex flex-1 overflow-hidden">
        {/* Sidebar doesn't need auth info */}
        <Sidebar activeTab={activeTab} setActiveTab={setActiveTab} />

        <div className="flex flex-1 flex-col overflow-hidden">
          {/* Topbar needs repo info and user info */}
          <Topbar
            userRepos={userRepos}
            selectedRepo={selectedRepo}
            setSelectedRepo={setSelectedRepo}
            userInfo={userInfo} // Pass fetched user info
          />
          {/* Main content area */}
          <div className="flex flex-1 overflow-hidden">
            {renderTabContent()}
          </div>
        </div>
      </div>
      {/* Bottombar likely doesn't need auth info */}
      <Bottombar />
    </div>
  );
}