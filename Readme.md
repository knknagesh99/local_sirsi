// ==UserScript==
// @name         Exam WebSocket Helper
// @namespace    local
// @version      1.0
// @description  Connect to the local exam helper and send application status messages
// @match        *://*/*
// @grant        none
// ==/UserScript==

(() => {
  "use strict";

  let socket;
  let serverToken;

  function show(id, display = "block") {
    const element = document.getElementById(id);
    if (element) element.style.display = display;
  }

  function create_hs(appl_no, authType) {
    const smartDiv = document.getElementById("smartLock");
    const faceAuth = document.getElementById("faceAuthReq");
    const proctorReq = document.getElementById("proctorReq");

    if (authType === "Anugyna") {
      socket = new WebSocket("ws://localhost:8000/");
      show("smartLogout");
    } else {
      const displayByType = {
        faceAuthReq: "faceAuthReq",
        NoExam: "smartNoExam",
        completeFlow: "smartcompleteFlow",
        scheduleDate: "scheduleDate",
        NOfaceAuthReq: "NoFaceAuthReq",
        bioAuth: "NoFaceAuthReq",
        attemptMaxtimes: "faceauthmaxtimeallowed",
        attemptMaxtimeDL: "smartfaceauthmaxtimeallowedDL",
        attemptMaxtimeslot: "faceauthmaxtimeallowedslot",
        alreadyCompleted: "alreadyCompleted",
        expiredApplication: "expiredApplication",
        photoUpload: "smartphotoUpload",
        capturePhoto: "capturePhoto",
        maxLogins: "maxLogins",
        quotaFlag: "quotaFlag"
      };

      if (displayByType[authType]) show(displayByType[authType]);

      [
        "photoUpload", "completeFlow", "NoExam",
        "faceauthmaxtimeallowedDL", "faceauthmaxtimeallowedslot",
        "faceAuthReq", "proctorReq"
      ].forEach(id => show(id, "none"));

      show("smartLogout");
    }

    // Only a WebSocket is created for Anugyna.
    if (!socket) return;

    socket.addEventListener("open", () => {
      if (authType !== "Anugyna" || appl_no == null) return;

      serverToken = "12345";
      socket.send(JSON.stringify({
        type: "Authentication",
        token: "1234",
        userid: appl_no
      }));

      if (smartDiv) smartDiv.style.display = "none";
      sessionStorage.setItem("authType", "Anugyna");
    });

    socket.addEventListener("message", event => {
      try {
        const response = JSON.parse(event.data);
        if (response.type === "Authentication") {
          const status =
            typeof SHA256 === "function" &&
            SHA256(serverToken) === response.token_hash
              ? "success"
              : "failed";
          console.log({ status }, "allSet");
        }
      } catch (error) {
        console.error("Invalid WebSocket message:", error);
      }
    });

    socket.addEventListener("close", () => {
      sessionStorage.clear();
      console.log("disconnected!");
      resetDisplay(smartDiv, faceAuth, proctorReq);
    });

    socket.addEventListener("error", error => {
      sessionStorage.clear();
      console.error("WebSocket error:", error);
      resetDisplay(smartDiv, faceAuth, proctorReq);
    });
  }

  function resetDisplay(smartDiv, faceAuth, proctorReq) {
    if (smartDiv) smartDiv.style.display = "block";
    if (faceAuth) faceAuth.style.display = "none";
    if (proctorReq) proctorReq.style.display = "none";

    [
      "smartLogout", "NoFaceAuthReq", "maxLogins", "capturePhoto",
      "photoUpload", "alreadyCompleted", "expiredApplication",
      "faceauthmaxtimeallowedslot", "faceauthmaxtimeallowedDL",
      "faceauthmaxtimeallowed", "scheduleDate", "completeFlow",
      "NoExam", "smartphotoUpload", "smartcompleteFlow", "smartNoExam",
      "smartfaceauthmaxtimeallowedDL", "smartfaceauthmaxtimeallowedslot",
      "faceAuthReq"
    ].forEach(id => show(id, "none"));
  }

  function applStatus(url, slot, appl_no) {
    // Supports applStatus(url) using the form value, or
    // applStatus(url, slot, appl_no) using explicit arguments.
    if (arguments.length === 1) {
      appl_no = document.forms.authentication?.elements.llappln?.value;
      slot = "";
    }

    const request = {
      type: "GenerateLicence",
      userid: appl_no,
      url: slot === "slots"
        ? `${location.origin}//${url}`
        : `${location.origin}${location.pathname.slice(
            0,
            location.pathname.lastIndexOf("/") + 1
          )}${url}`
    };

    setTimeout(() => {
      if (socket?.readyState === WebSocket.OPEN) {
        socket.send(JSON.stringify(request));
      } else {
        console.warn("WebSocket is not connected.");
      }
    }, arguments.length === 1 ? 0 : 1000);
  }

  function logout() {
    setTimeout(() => {
      if (socket?.readyState === WebSocket.OPEN) {
        socket.send(JSON.stringify({ type: "LOGOUT" }));
      }
    }, 1000);
  }

  // Expose functions for page buttons/handlers.
  window.create_hs = create_hs;
  window.applStatus = applStatus;
  window.logout = logout;
})();
