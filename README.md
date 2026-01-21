import React, { useState, useEffect } from "react";
import { base44 } from "@/api/base44Client";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import PasswordSetup from "../components/secure-folder/PasswordSetup";
import SecurityQuestionsSetup from "../components/secure-folder/SecurityQuestionsSetup";
import LoginForm from "../components/secure-folder/LoginForm";
import FileVault from "../components/secure-folder/FileVault";
import PasswordReset from "../components/secure-folder/PasswordReset";

// Simple hash function for password (in production, use proper encryption)
const hashPassword = async (password) => {
  const encoder = new TextEncoder();
  const data = encoder.encode(password);
  const hashBuffer = await crypto.subtle.digest('SHA-256', data);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  const hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
  return hashHex;
};

export default function SecureFolder() {
  const [appState, setAppState] = useState("loading"); // loading, setup_password, setup_security, login, vault, reset
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [isUploading, setIsUploading] = useState(false);
  const [verifiedForReset, setVerifiedForReset] = useState(false);
  const queryClient = useQueryClient();

  const { data: currentUser, isLoading: userLoading } = useQuery({
    queryKey: ['currentUser'],
    queryFn: () => base44.auth.me(),
  });

  const { data: files, isLoading: filesLoading } = useQuery({
    queryKey: ['secureFiles'],
    queryFn: () => base44.entities.SecureFile.filter({ created_by: currentUser?.email }),
    enabled: isAuthenticated && !!currentUser?.email,
    initialData: [],
  });

  useEffect(() => {
    if (!userLoading && currentUser) {
      // Check if user has password set up
      if (!currentUser.vault_password_hash) {
        setAppState("setup_password");
      }
      // Check if user has security questions set up
      else if (!currentUser.security_answer_school_hash || 
               !currentUser.security_answer_city_hash || 
               !currentUser.security_answer_food_hash) {
        setAppState("setup_security");
      }
      // Everything is set up, show login
      else {
        setAppState("login");
      }
    }
  }, [currentUser, userLoading]);

  const handlePasswordSet = async (password) => {
    const passwordHash = await hashPassword(password);
    await base44.auth.updateMe({
      vault_password_hash: passwordHash,
    });
    queryClient.invalidateQueries({ queryKey: ['currentUser'] });
    setAppState("setup_security");
  };

  const handleSecurityQuestionsSet = async ({ schoolName, city, food }) => {
    const schoolHash = await hashPassword(schoolName);
    const cityHash = await hashPassword(city);
    const foodHash = await hashPassword(food);
    
    await base44.auth.updateMe({
      security_answer_school_hash: schoolHash,
      security_answer_city_hash: cityHash,
      security_answer_food_hash: foodHash,
    });
    
    queryClient.invalidateQueries({ queryKey: ['currentUser'] });
    setIsAuthenticated(true);
    setAppState("vault");
  };

  const handleLogin = async (password) => {
    const hash = await hashPassword(password);
    if (hash === currentUser.vault_password_hash) {
      setIsAuthenticated(true);
      setAppState("vault");
      return true;
    }
    return false;
  };

  const handlePasswordReset = async (data) => {
    if (data.step === 'verify') {
      const schoolHash = await hashPassword(data.schoolName);
      const cityHash = await hashPassword(data.city);
      const foodHash = await hashPassword(data.food);
      
      const isValid = 
        schoolHash === currentUser.security_answer_school_hash &&
        cityHash === currentUser.security_answer_city_hash &&
        foodHash === currentUser.security_answer_food_hash;
      
      if (isValid) {
        setVerifiedForReset(true);
      }
      
      return isValid;
    } else if (data.step === 'reset' && verifiedForReset) {
      const hash = await hashPassword(data.newPassword);
      await base44.auth.updateMe({ vault_password_hash: hash });
      queryClient.invalidateQueries({ queryKey: ['currentUser'] });
      setVerifiedForReset(false);
      setIsAuthenticated(true);
      setAppState("vault");
    }
  };

  const handleUpload = async (uploadedFiles) => {
    setIsUploading(true);
    
    for (const file of uploadedFiles) {
      const { file_url } = await base44.integrations.Core.UploadFile({ file });
      
      await base44.entities.SecureFile.create({
        file_name: file.name,
        file_url: file_url,
        file_type: file.type,
        file_size: file.size,
        uploaded_date: new Date().toISOString(),
      });
    }

    queryClient.invalidateQueries({ queryKey: ['secureFiles'] });
    setIsUploading(false);
  };

  const handleDelete = async (fileId) => {
    await base44.entities.SecureFile.delete(fileId);
    queryClient.invalidateQueries({ queryKey: ['secureFiles'] });
  };

  const handleLogout = () => {
    setIsAuthenticated(false);
    setAppState("login");
  };

  if (appState === "loading" || userLoading) {
    return (
      <div className="flex items-center justify-center min-h-screen">
        <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-blue-600"></div>
      </div>
    );
  }

  if (appState === "setup_password") {
    return <PasswordSetup onPasswordSet={handlePasswordSet} />;
  }

  if (appState === "setup_security") {
    return <SecurityQuestionsSetup onComplete={handleSecurityQuestionsSet} />;
  }

  if (appState === "login") {
    return (
      <LoginForm
        onLogin={handleLogin}
        onForgotPassword={() => setAppState("reset")}
      />
    );
  }

  if (appState === "reset") {
    return (
      <PasswordReset
        onPasswordReset={handlePasswordReset}
        onCancel={() => {
          setAppState("login");
          setVerifiedForReset(false);
        }}
      />
    );
  }

  if (appState === "vault" && isAuthenticated) {
    return (
      <FileVault
        files={files}
        onUpload={handleUpload}
        onDelete={handleDelete}
        onLogout={handleLogout}
        isUploading={isUploading}
      />
    );
  }

  return null;
}
