<script lang="ts">
  import { onMount } from 'svelte';
  import cordeLogo from './assets/corde icon.png';

  const TXT = "refactor the auth middleware to use JWT refresh tokens, add per-user rate limiting and write tests";
  
  let txContent = "Press Ctrl+Shift+V and start speaking";
  let activeKeys = [false, false, false]; // Ctrl, Shift, V
  let micActive = false;
  let micBarHeights = [6, 6, 6, 6, 6];
  let eqBars = Array(22).fill(0).map((_, i) => 14 + Math.abs(Math.sin(i * 0.8)) * 40);
  let micInterval: number;

  const IC: Record<string, string> = {
    "Claude Code": "m4.7144 15.9555 4.7174-2.6471.079-.2307-.079-.1275h-.2307l-.7893-.0486-2.6956-.0729-2.3375-.0971-2.2646-.1214-.5707-.1215-.5343-.7042.0546-.3522.4797-.3218.686.0608 1.5179.1032 2.2767.1578 1.6514.0972 2.4468.255h.3886l.0546-.1579-.1336-.0971-.1032-.0972L6.973 9.8356l-2.55-1.6879-1.3356-.9714-.7225-.4918-.3643-.4614-.1578-1.0078.6557-.7225.8803.0607.2246.0607.8925.686 1.9064 1.4754 2.4893 1.8336.3643.3035.1457-.1032.0182-.0728-.164-.2733-1.3539-2.4467-1.445-2.4893-.6435-1.032-.17-.6194c-.0607-.255-.1032-.4674-.1032-.7285L6.287.1335 6.6997 0l.9957.1336.419.3642.6192 1.4147 1.0018 2.2282 1.5543 3.0296.4553.8985.2429.8318.091.255h.1579v-.1457l.1275-1.706.2368-2.0947.2307-2.6957.0789-.7589.3764-.9107.7468-.4918.5828.2793.4797.686-.0668.4433-.2853 1.8517-.5586 2.9021-.3643 1.9429h.2125l.2429-.2429.9835-1.3053 1.6514-2.0643.7286-.8196.85-.9046.5464-.4311h1.0321l.759 1.1293-.34 1.1657-1.0625 1.3478-.8804 1.1414-1.2628 1.7-.7893 1.36.0729.1093.1882-.0183 2.8535-.607 1.5421-.2794 1.8396-.3157.8318.3886.091.3946-.3278.8075-1.967.4857-2.3072.4614-3.4364.8136-.0425.0304.0486.0607 1.5482.1457.6618.0364h1.621l3.0175.2247.7892.522.4736.6376-.079.4857-1.2142.6193-1.6393-.3886-3.825-.9107-1.3113-.3279h-.1822v.1093l1.0929 1.0686 2.0035 1.8092 2.5075 2.3314.1275.5768-.3218.4554-.34-.0486-2.2039-1.6575-.85-.7468-1.9246-1.621h-.1275v.17l.4432.6496 2.3436 3.5214.1214 1.0807-.17.3521-.6071.2125-.6679-.1214-1.3721-1.9246L14.38 17.959l-1.1414-1.9428-.1397.079-.674 7.2552-.3156.3703-.7286.2793-.6071-.4614-.3218-.7468.3218-1.4753.3886-1.9246.3157-1.53.2853-1.9004.17-.6314-.0121-.0425-.1397.0182-1.4328 1.9672-2.1796 2.9446-1.7243 1.8456-.4128.164-.7164-.3704.0667-.6618.4008-.5889 2.386-3.0357 1.4389-1.882.929-1.0868-.0062-.1579h-.0546l-6.3385 4.1164-1.1293.1457-.4857-.4554.0608-.7467.2307-.2429 1.9064-1.3114Z",
    "OpenCode": "M22 24H2V0h20zM17 4.8H7v14.4h10z",
    "Gemini CLI": "M11.04 19.32Q12 21.51 12 24q0-2.49.93-4.68.96-2.19 2.58-3.81t3.81-2.55Q21.51 12 24 12q-2.49 0-4.68-.93a12.3 12.3 0 0 1-3.81-2.58 12.3 12.3 0 0 1-2.58-3.81Q12 2.49 12 0q0 2.49-.96 4.68-.93 2.19-2.55 3.81a12.3 12.3 0 0 1-3.81 2.58Q2.49 12 0 12q2.49 0 4.68.96 2.19.93 3.81 2.55t2.55 3.81",
    "Amp": "M12 0c6.628 0 12 5.373 12 12s-5.372 12-12 12C5.373 24 0 18.627 0 12S5.373 0 12 0zm-.92 19.278l5.034-8.377a.444.444 0 00.097-.268.455.455 0 00-.455-.455l-2.851.004.924-5.468-.927-.003-5.018 8.367s-.1.183-.1.291c0 .251.204.455.455.455l2.831-.004-.901 5.458z",
    "Cursor": "M11.503.131 1.891 5.678a.84.84 0 0 0-.42.726v11.188c0 .3.162.575.42.724l9.609 5.55a1 1 0 0 0 .998 0l9.61-5.55a.84.84 0 0 0 .42-.724V6.404a.84.84 0 0 0-.42-.726L12.497.131a1.01 1.01 0 0 0-.996 0M2.657 6.338h18.55c.263 0 .43.287.297.515L12.23 22.918c-.062.107-.229.064-.229-.06V12.335a.59.59 0 0 0-.295-.51l-9.11-5.257c-.109-.063-.064-.23.061-.23",
    "Copilot CLI": "M23.922 16.997C23.061 18.492 18.063 22.02 12 22.02 5.937 22.02.939 18.492.078 16.997A.641.641 0 0 1 0 16.741v-2.869a.883.883 0 0 1 .053-.22c.372-.935 1.347-2.292 2.605-2.656.167-.429.414-1.055.644-1.517a10.098 10.098 0 0 1-.052-1.086c0-1.331.282-2.499 1.132-3.368.397-.406.89-.717 1.474-.952C7.255 2.937 9.248 1.98 11.978 1.98c2.731 0 4.767.957 6.166 2.093.584.235 1.077.546 1.474.952.85.869 1.132 2.037 1.132 3.368 0 .368-.014.733-.052 1.086.23.462.477 1.088.644 1.517 1.258.364 2.233 1.721 2.605 2.656a.841.841 0 0 1 .053.22v2.869a.641.641 0 0 1-.078.256Zm-11.75-5.992h-.344a4.359 4.359 0 0 1-.355.508c-.77.947-1.918 1.492-3.508 1.492-1.725 0-2.989-.359-3.782-1.259a2.137 2.137 0 0 1-.085-.104L4 11.746v6.585c1.435.779 4.514 2.179 8 2.179 3.486 0 6.565-1.4 8-2.179v-6.585l-.098-.104s-.033.045-.085.104c-.793.9-2.057 1.259-3.782 1.259-1.59 0-2.738-.545-3.508-1.492a4.359 4.359 0 0 1-.355-.508Zm2.328 3.25c.549 0 1 .451 1 1v2c0 .549-.451 1-1 1-.549 0-1-.451-1-1v-2c0-.549.451-1 1-1Zm-5 0c.549 0 1 .451 1 1v2c0 .549-.451 1-1 1-.549 0-1-.451-1-1v-2c0-.549.451-1 1-1Zm3.313-6.185c.136 1.057.403 1.913.878 2.497.442.544 1.134.938 2.344.938 1.573 0 2.292-.337 2.657-.751.384-.435.558-1.15.558-2.361 0-1.14-.243-1.847-.705-2.319-.477-.488-1.319-.862-2.824-1.025-1.487-.161-2.192.138-2.533.529-.269.307-.437.808-.438 1.578v.021c0 .265.021.562.063.893Zm-1.626 0c.042-.331.063-.628.063-.894v-.02c-.001-.77-.169-1.271-.438-1.578-.341-.391-1.046-.69-2.533-.529-1.505.163-2.347.537-2.824 1.025-.462.472-.705 1.179-.705 2.319 0 1.211.175 1.926.558 2.361.365.414 1.084.751 2.657.751 1.21 0 1.902-.394 2.344-.938.475-.584.742-1.44.878-2.497Z",
    "Cline": "m23.365 13.556-1.442-2.895V8.994c0-2.764-2.218-5.002-4.954-5.002h-2.464c.178-.367.276-.779.276-1.213A2.77 2.77 0 0 0 12.018 0a2.77 2.77 0 0 0-2.763 2.779c0 .434.098.846.276 1.213H7.067c-2.736 0-4.954 2.238-4.954 5.002v1.667L.64 13.549c-.149.29-.149.636 0 .927l1.472 2.855v1.667C2.113 21.762 4.33 24 7.067 24h9.902c2.736 0 4.954-2.238 4.954-5.002V17.33l1.44-2.865c.143-.286.143-.622.002-.91m-12.854 2.36a2.27 2.27 0 0 1-2.261 2.273 2.27 2.27 0 0 1-2.261-2.273v-4.042A2.27 2.27 0 0 1 8.249 9.6a2.267 2.267 0 0 1 2.262 2.274zm7.285 0a2.27 2.27 0 0 1-2.26 2.273 2.27 2.27 0 0 1-2.262-2.273v-4.042A2.267 2.267 0 0 1 15.535 9.6a2.267 2.267 0 0 1 2.261 2.274",
    "GNOME Terminal": "M1.846 0A1.841 1.841 0 000 1.846v18.463c0 1.022.823 1.845 1.846 1.845h20.308A1.841 1.841 0 0024 20.31V1.846A1.841 1.841 0 0022.154 0H1.846zm0 .924h20.308c.512 0 .922.41.922.922v18.463c0 .511-.41.921-.922.921H1.846a.919.919 0 01-.922-.921V1.846c0-.512.41-.922.922-.922zm0 .922v18.463h20.308V1.846H1.846zm1.845 2.14l3.235 1.758v.836L3.69 8.477V7.385l2.243-1.207v-.033L3.69 5.076v-1.09zM7.846 9.23h3.693v.924H7.846V9.23zM0 21.736v.418C0 23.177.823 24 1.846 24h20.308A1.841 1.841 0 0024 22.154v-.418a2.334 2.334 0 01-1.846.918H1.846A2.334 2.334 0 010 21.736Z",
    "iTerm2": "M24 5.359v13.282A5.36 5.36 0 0 1 18.641 24H5.359A5.36 5.36 0 0 1 0 18.641V5.359A5.36 5.36 0 0 1 5.359 0h13.282A5.36 5.36 0 0 1 24 5.359m-.932-.233A4.196 4.196 0 0 0 18.874.932H5.126A4.196 4.196 0 0 0 .932 5.126v13.748a4.196 4.196 0 0 0 4.194 4.194h13.748a4.196 4.196 0 0 0 4.194-4.194zm-.816.233v13.282a3.613 3.613 0 0 1-3.611 3.611H5.359a3.613 3.613 0 0 1-3.611-3.611V5.359a3.613 3.613 0 0 1 3.611-3.611h13.282a3.613 3.613 0 0 1 3.611 3.611M8.854 4.194v6.495h.962V4.194zM5.483 9.493v1.085h.597V9.48q.283-.037.508-.133.373-.165.575-.448.208-.284.208-.649a.9.9 0 0 0-.171-.568 1.4 1.4 0 0 0-.426-.388 3 3 0 0 0-.544-.261 32 32 0 0 0-.545-.209 1.8 1.8 0 0 1-.426-.216q-.164-.12-.164-.284 0-.223.179-.351.18-.126.485-.127.344 0 .575.105.239.105.5.298l.433-.5a2.3 2.3 0 0 0-.605-.433 1.6 1.6 0 0 0-.582-.159v-.968h-.597v.978a2 2 0 0 0-.477.127 1.2 1.2 0 0 0-.545.411q-.194.268-.194.634 0 .335.164.56.164.224.418.38a4 4 0 0 0 .552.262q.291.104.545.209.261.104.425.238a.39.39 0 0 1 .165.321q0 .225-.187.359-.18.134-.537.134-.381 0-.717-.134a4.4 4.4 0 0 1-.649-.351l-.388.589q.209.173.477.306.276.135.575.217.191.046.373.064",
    "Warp": "M12.035 2.723h9.253A2.712 2.712 0 0 1 24 5.435v10.529a2.712 2.712 0 0 1-2.712 2.713H8.047Zm-1.681 2.6L6.766 19.677h5.598l-.399 1.6H2.712A2.712 2.712 0 0 1 0 18.565V8.036a2.712 2.712 0 0 1 2.712-2.712Z",
    "Ghostty": "M12 0C6.7 0 2.4 4.3 2.4 9.6v11.146c0 1.772 1.45 3.267 3.222 3.254a3.18 3.18 0 0 0 1.955-.686 1.96 1.96 0 0 1 2.444 0 3.18 3.18 0 0 0 1.976.686c.75 0 1.436-.257 1.98-.686.715-.563 1.71-.587 2.419-.018.59.476 1.355.743 2.182.699 1.705-.094 3.022-1.537 3.022-3.244V9.601C21.6 4.3 17.302 0 12 0M6.069 6.562a1 1 0 0 1 .46.131l3.578 2.065v.002a.974.974 0 0 1 0 1.687L6.53 12.512a.975.975 0 0 1-.976-1.687L7.67 9.602 5.553 8.38a.975.975 0 0 1 .515-1.818m7.438 2.063h4.7a.975.975 0 1 1 0 1.95h-4.7a.975.975 0 0 1 0-1.95",
    "Alacritty": "m10.065 0-8.57 21.269h3.595l6.91-16.244 6.91 16.244h3.594l-8.57-21.269zm1.935 9.935c-.76666 1.8547-1.5334 3.7094-2.298 5.565 1.475 4.54 1.475 4.54 2.298 8.5.823-3.96.823-3.96 2.297-8.5-.76637-1.8547-1.5315-3.7099-2.297-5.565z",
    "Neovim": "M2.214 4.954v13.615L7.655 24V10.314L3.312 3.845 2.214 4.954zm4.999 17.98-4.557-4.548V5.136l.59-.596 3.967 5.908v12.485zm14.573-4.457-.862.937-4.24-6.376V0l5.068 5.092.034 13.385zM7.431.001l12.998 19.835-3.637 3.637L3.787 3.683 7.43 0z",
    "Zed": "M2.25 1.5a.75.75 0 0 0-.75.75v16.5H0V2.25A2.25 2.25 0 0 1 2.25 0h20.095c1.002 0 1.504 1.212.795 1.92L10.764 14.298h3.486V12.75h1.5v1.922a1.125 1.125 0 0 1-1.125 1.125H9.264l-2.578 2.578h11.689V9h1.5v9.375a1.5 1.5 0 0 1-1.5 1.5H5.185L2.562 22.5H21.75a.75.75 0 0 0 .75-.75V5.25H24v16.5A2.25 2.25 0 0 1 21.75 24H1.655C.653 24 .151 22.788.86 22.08L13.19 9.75H9.75v1.5h-1.5V9.375A1.125 1.125 0 0 1 9.375 8.25h5.314l2.625-2.625H5.625V15h-1.5V5.625a1.5 1.5 0 0 1 1.5-1.5h13.19L21.438 1.5z",
    "JetBrains": "M2.345 23.997A2.347 2.347 0 0 1 0 21.652V10.988C0 9.665.535 8.37 1.473 7.433l5.965-5.961A5.01 5.01 0 0 1 10.989 0h10.666A2.347 2.347 0 0 1 24 2.345v10.664a5.056 5.056 0 0 1-1.473 3.554l-5.965 5.965A5.017 5.017 0 0 1 13.007 24v-.003H2.345Zm8.969-6.854H5.486v1.371h5.828v-1.371ZM3.963 6.514h13.523v13.519l4.257-4.257a3.936 3.936 0 0 0 1.146-2.767V2.345c0-.678-.552-1.234-1.234-1.234H10.989a3.897 3.897 0 0 0-2.767 1.145L3.963 6.514Zm-.192.192L2.256 8.22a3.944 3.944 0 0 0-1.145 2.768v10.664c0 .678.552 1.234 1.234 1.234h10.666a3.9 3.9 0 0 0 2.767-1.146l1.512-1.511H3.771V6.706Z",
    "tmux": "M24 2.251V10.5H12.45V0h9.3A2.251 2.251 0 0 1 24 2.251zM12.45 11.4H24v10.5h-.008A2.25 2.25 0 0 1 21.75 24H2.25a2.247 2.247 0 0 1-2.242-2.1H0V2.251A2.251 2.251 0 0 1 2.25 0h9.3v21.6h.9V11.4zm11.242 10.5H.308a1.948 1.948 0 0 0 1.942 1.8h19.5a1.95 1.95 0 0 0 1.942-1.8z"
  };

  const R1 = ["Claude Code", "OpenCode", "Gemini CLI", "Aider", "Goose", "Amp", "Cursor", "Copilot CLI", "Cline"];
  const R2 = ["GNOME Terminal", "iTerm2", "Warp", "Ghostty", "Alacritty", "kitty", "VS Code", "Windows Terminal", "Neovim", "Zed", "JetBrains", "tmux"];

  const sleep = (ms: number) => new Promise(r => setTimeout(r, ms));

  function wv(on: boolean) {
    clearInterval(micInterval);
    if (!on) {
      micBarHeights = [6, 6, 6, 6, 6];
      return;
    }
    micInterval = setInterval(() => {
      micBarHeights = micBarHeights.map(() => 5 + Math.random() * 16);
    }, 90);
  }

  async function loop() {
    if (window.matchMedia("(prefers-reduced-motion: reduce)").matches) {
      txContent = TXT;
      return;
    }
    while (true) {
      txContent = "Press Ctrl+Shift+V and start speaking";
      await sleep(1400);
      for (let i = 0; i < activeKeys.length; i++) {
        activeKeys[i] = true;
        await sleep(130);
      }
      txContent = "";
      wv(true);
      await sleep(1500);
      activeKeys = [false, false, false];
      wv(false);
      for (let i = 1; i <= TXT.length; i++) {
        txContent = TXT.slice(0, i);
        await sleep(14);
      }
      await sleep(3200);
    }
  }

  let recommendedOS = "";
  let mobileMenuOpen = false;
  let navHidden = false;
  let billingSuccess = false;
  let downloadGuideModal: "mac" | "win" | null = null;
  let copiedXattrCommand = false;

  function openDownloadGuide(os: "mac" | "win") {
    downloadGuideModal = os;
    copiedXattrCommand = false;
  }

  function closeDownloadGuide() {
    downloadGuideModal = null;
  }

  async function copyMacTerminalCommand() {
    try {
      await navigator.clipboard.writeText("xattr -cr /Applications/Cordė.app");
      copiedXattrCommand = true;
      setTimeout(() => {
        copiedXattrCommand = false;
      }, 2200);
    } catch {
      // ignore clipboard errors
    }
  }
  let emailConfirmation: {
    state: "success" | "error";
    appUrl: string | null;
    mobile: boolean;
  } | null = null;

  onMount(() => {
    const previousScrollRestoration = history.scrollRestoration;
    history.scrollRestoration = "manual";
    document.documentElement.scrollTop = 0;
    document.body.scrollTop = 0;

    if (window.location.pathname.replace(/\/+$/, "") === "/auth/confirmed") {
      const confirmationUrl = new URL(window.location.href);
      const code = confirmationUrl.searchParams.get("code");
      const failed = ["error", "error_code", "error_description"].some((key) =>
        confirmationUrl.searchParams.has(key)
      );
      const mobile = /Android|iPhone|iPad|iPod/i.test(navigator.userAgent);
      let appUrl: string | null = null;
      if (code) {
        const callbackUrl = new URL("corde://auth/callback");
        callbackUrl.searchParams.set("code", code);
        appUrl = callbackUrl.toString();
      }
      emailConfirmation = {
        state: code && !failed ? "success" : "error",
        appUrl,
        mobile,
      };
      document.title = emailConfirmation.state === "success"
        ? "Email confirmed | Cordė"
        : "Confirmation link issue | Cordė";
      return () => {
        history.scrollRestoration = previousScrollRestoration;
      };
    }

    billingSuccess = new URLSearchParams(window.location.search).get("billing") === "success";
    if (billingSuccess) {
      document.title = "Payment received | Cordė";
      return () => {
        history.scrollRestoration = previousScrollRestoration;
      };
    }

    loop();
    const u = navigator.userAgent;
    recommendedOS = /Android|iPhone|iPad/.test(u) ? "" : /Mac/.test(u) ? "mac" : /Win/.test(u) ? "win" : /Linux|X11/.test(u) ? "linux" : "";

    let previousScrollY = window.scrollY;
    const handleScroll = () => {
      const currentScrollY = window.scrollY;
      if (currentScrollY < 80 || currentScrollY < previousScrollY) {
        navHidden = false;
      } else if (currentScrollY > previousScrollY) {
        navHidden = true;
      }
      previousScrollY = currentScrollY;
    };
    window.addEventListener("scroll", handleScroll, { passive: true });

    const cards = document.querySelectorAll("#features .card");
    let featureObserver: IntersectionObserver | undefined;
    if ("IntersectionObserver" in window) {
      featureObserver = new IntersectionObserver(entries => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            entry.target.classList.add("in");
            featureObserver?.unobserve(entry.target);
          }
        });
      }, { threshold: 0.25 });
      cards.forEach(card => featureObserver?.observe(card));
    } else {
      cards.forEach(card => card.classList.add("in"));
    }

    return () => {
      window.removeEventListener("scroll", handleScroll);
      featureObserver?.disconnect();
      history.scrollRestoration = previousScrollRestoration;
    };
  });

  function getChipBackground(i: number) {
    return `linear-gradient(135deg, hsl(${(i * 43 + 235) % 360} 70% 58%), hsl(${(i * 43 + 265) % 360} 70% 38%))`;
  }
</script>

<div class="glow" aria-hidden="true"></div><div class="haze" aria-hidden="true"></div>
<div class="wrap">
  {#if emailConfirmation}
    <main class="billing-success" aria-live="polite">
      <div class="billing-success-card">
        <div class="billing-success-icon" aria-hidden="true">
          {emailConfirmation.state === "success" ? "✓" : "!"}
        </div>
        <p class="billing-success-eyebrow">
          {emailConfirmation.state === "success" ? "EMAIL CONFIRMED" : "CONFIRMATION LINK ISSUE"}
        </p>
        {#if emailConfirmation.state === "success"}
          <h1>Your email is confirmed.</h1>
          <p class="billing-success-copy">
            {emailConfirmation.mobile
              ? "Your Cordė account is ready. Open Cordė on your laptop and sign in with this email to continue."
              : "Your Cordė account is ready. Continue in the desktop app to finish signing in."}
          </p>
          {#if emailConfirmation.appUrl && !emailConfirmation.mobile}
            <a class="btn lg" href={emailConfirmation.appUrl}>Open Cordė desktop app</a>
          {/if}
          <p class="billing-success-note">
            {emailConfirmation.mobile
              ? "You can close this page and return to your laptop. Your account will be ready when you sign in to Cordė."
              : "If the app doesn't open, launch Cordė on your laptop and sign in with the email address you just confirmed."}
          </p>
        {:else}
          <h1>This confirmation link couldn't be used.</h1>
          <p class="billing-success-copy">
            The link may have expired or already been used. Return to Cordė on your laptop and request a new confirmation email.
          </p>
        {/if}
        <a class="billing-success-home" href="/">Return to Cordė website</a>
      </div>
    </main>
  {:else if billingSuccess}
    <main class="billing-success" aria-live="polite">
      <div class="billing-success-card">
        <div class="billing-success-icon" aria-hidden="true">✓</div>
        <p class="billing-success-eyebrow">PAYMENT RECEIVED</p>
        <h1>You're all set.</h1>
        <p class="billing-success-copy">
          Your payment was successful. Open Cordė to refresh your plan.
        </p>
        <a class="btn lg" href="corde://open">Open Cordė desktop app</a>
        <p class="billing-success-note">
          In the app, open <strong>Settings → Billing</strong> and select <strong>Refresh status</strong>.
          If your plan doesn't appear yet, wait a moment and refresh again.
        </p>
        <a class="billing-success-home" href="/">Return to Cordė website</a>
      </div>
    </main>
  {:else}
  <nav class:hidden={navHidden}>
    <a class="brand" href="#top" aria-label="Cordė home">
      <img src={cordeLogo} alt="" aria-hidden="true" />Cordė
    </a>
    <div class="links" class:open={mobileMenuOpen}>
      <a href="#features" onclick={() => mobileMenuOpen = false}>Product</a>
      <a href="#teams" onclick={() => mobileMenuOpen = false}>Use cases</a>
      <a href="#pricing" onclick={() => mobileMenuOpen = false}>Pricing</a>
      <a href="#faq" onclick={() => mobileMenuOpen = false}>FAQ</a>
    </div>
    <div class="nav-actions">
      <a class="btn" href="#pricing">View pricing</a>
      <button
        class="menu-toggle"
        type="button"
        aria-label={mobileMenuOpen ? "Close navigation menu" : "Open navigation menu"}
        aria-expanded={mobileMenuOpen}
        onclick={() => mobileMenuOpen = !mobileMenuOpen}
      >
        {#if mobileMenuOpen}
          <svg viewBox="0 0 24 24" aria-hidden="true"><path d="m6 6 12 12M18 6 6 18"/></svg>
        {:else}
          <svg viewBox="0 0 24 24" aria-hidden="true"><path d="M4 7h16M4 12h16M4 17h16"/></svg>
        {/if}
      </button>
    </div>
  </nav>
  <div class="nav-spacer" aria-hidden="true"></div>

  <header class="hero" id="top">
    <h1>Smart Voice Input for Your Terminal</h1>
    <p class="sub">Speak your prompts and Cordė types them into any terminal or window. Private, instant, and processed entirely on your machine.</p>
    <div class="box" aria-label="Animated demo: speech typed into a terminal prompt">
      <div class="keys">
        <span class="k" class:dn={activeKeys[0]}>Ctrl</span>
        <span class="k" class:dn={activeKeys[1]}>Shift</span>
        <span class="k" class:dn={activeKeys[2]}>V</span>
      </div>
      <div class="ph"><b>&gt;</b> <span>{txContent}</span><span class="cur"></span></div>
      <div class="mic">
        {#each micBarHeights as height}
          <i style="height: {height}px"></i>
        {/each}
      </div>
      <span class="btn go">Send</span>
    </div>
    <div class="cta">
      <a class="btn lg" href="#pricing">See pricing</a>
      <a class="btn lg ghost" href="#how">See how it works</a>
    </div>
    <a class="trial-card" href="#download">
      <span class="trial-card-copy">
        <b>Try Cordė free</b>
        <small>7-day free trial · No credit card required</small>
      </span>
      <span class="trial-card-arrow" aria-hidden="true">→</span>
    </a>
  </header>

  <div class="dl" id="download">
    <a
      class="dlb"
      href="https://github.com/corde-voice/corde-releases/releases/latest/download/Corde-macos-aarch64.dmg"
      class:rec={recommendedOS === 'mac'}
      data-os="mac"
      onclick={() => openDownloadGuide('mac')}
    >
      <svg class="fi" viewBox="0 0 24 24" aria-hidden="true"><path d="M12.152 6.896c-.948 0-2.415-1.078-3.96-1.04-2.04.027-3.91 1.183-4.961 3.014-2.117 3.675-.546 9.103 1.519 12.09 1.013 1.454 2.208 3.09 3.792 3.039 1.52-.065 2.09-.987 3.935-.987 1.831 0 2.35.987 3.96.948 1.637-.026 2.676-1.48 3.676-2.948 1.156-1.688 1.636-3.325 1.662-3.415-.039-.013-3.182-1.221-3.22-4.857-.026-3.04 2.48-4.494 2.597-4.559-1.429-2.09-3.623-2.324-4.39-2.376-2-.156-3.675 1.09-4.61 1.09zM15.53 3.83c.843-1.012 1.4-2.427 1.245-3.83-1.207.052-2.662.805-3.532 1.818-.78.896-1.454 2.338-1.273 3.714 1.338.104 2.715-.688 3.559-1.701"/></svg>
      <span><b>macOS</b><small>Download · Apple Silicon &amp; Intel</small></span>
    </a>
    <a
      class="dlb"
      href="https://github.com/corde-voice/corde-releases/releases/latest/download/Corde-setup.exe"
      class:rec={recommendedOS === 'win'}
      data-os="win"
      onclick={() => openDownloadGuide('win')}
    >
      <svg viewBox="0 0 24 24" fill="none" stroke="#fff" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><rect x="3" y="4" width="18" height="12" rx="2"/><path d="M8 20h8M12 16v4"/></svg>
      <span><b>Windows</b><small>Download · Windows 10 &amp; 11</small></span>
    </a>
    <span class="dlb" aria-disabled="true" data-os="linux">
      <svg class="fi" viewBox="0 0 24 24" aria-hidden="true"><path d="M12.504 0c-.155 0-.315.008-.48.021-4.226.333-3.105 4.807-3.17 6.298-.076 1.092-.3 1.953-1.05 3.02-.885 1.051-2.127 2.75-2.716 4.521-.278.832-.41 1.684-.287 2.489a.424.424 0 00-.11.135c-.26.268-.45.6-.663.839-.199.199-.485.267-.797.4-.313.136-.658.269-.864.68-.09.189-.136.394-.132.602 0 .199.027.4.055.536.058.399.116.728.04.97-.249.68-.28 1.145-.106 1.484.174.334.535.47.94.601.81.2 1.91.135 2.774.6.926.466 1.866.67 2.616.47.526-.116.97-.464 1.208-.946.587-.003 1.23-.269 2.26-.334.699-.058 1.574.267 2.577.2.025.134.063.198.114.333l.003.003c.391.778 1.113 1.132 1.884 1.071.771-.06 1.592-.536 2.257-1.306.631-.765 1.683-1.084 2.378-1.503.348-.199.629-.469.649-.853.023-.4-.2-.811-.714-1.376v-.097l-.003-.003c-.17-.2-.25-.535-.338-.926-.085-.401-.182-.786-.492-1.046h-.003c-.059-.054-.123-.067-.188-.135a.357.357 0 00-.19-.064c.431-1.278.264-2.55-.173-3.694-.533-1.41-1.465-2.638-2.175-3.483-.796-1.005-1.576-1.957-1.56-3.368.026-2.152.236-6.133-3.544-6.139zm.529 3.405h.013c.213 0 .396.062.584.198.19.135.33.332.438.533.105.259.158.459.166.724 0-.02.006-.04.006-.06v.105a.086.086 0 01-.004-.021l-.004-.024a1.807 1.807 0 01-.15.706.953.953 0 01-.213.335.71.71 0 00-.088-.042c-.104-.045-.198-.064-.284-.133a1.312 1.312 0 00-.22-.066c.05-.06.146-.133.183-.198.053-.128.082-.264.088-.402v-.02a1.21 1.21 0 00-.061-.4c-.045-.134-.101-.2-.183-.333-.084-.066-.167-.132-.267-.132h-.016c-.093 0-.176.03-.262.132a.8.8 0 00-.205.334 1.18 1.18 0 00-.09.4v.019c.002.089.008.179.02.267-.193-.067-.438-.135-.607-.202a1.635 1.635 0 01-.018-.2v-.02a1.772 1.772 0 01.15-.768c.082-.22.232-.406.43-.533a.985.985 0 01.594-.2zm-2.962.059h.036c.142 0 .27.048.399.135.146.129.264.288.344.465.09.199.14.4.153.667v.004c.007.134.006.2-.002.266v.08c-.03.007-.056.018-.083.024-.152.055-.274.135-.393.2.012-.09.013-.18.003-.267v-.015c-.012-.133-.04-.2-.082-.333a.613.613 0 00-.166-.267.248.248 0 00-.183-.064h-.021"/></svg>
      <span><b>Linux</b><small>Coming soon · AppImage, deb, rpm</small></span>
    </span>
  </div>

  <div class="works">
    <p>Works with every coding agent and terminal</p>
    <div class="mq" id="mq1">
      <div class="tr">
        <div class="set">
          {#each R1 as n, i}
            <div class="chip">
              {#if IC[n]}
                <svg viewBox="0 0 24 24" width="22" height="22" fill="currentColor" aria-hidden="true"><path d={IC[n]}/></svg>
              {:else}
                <b style="background: {getChipBackground(i)}">{n[0].toUpperCase()}</b>
              {/if}
              {n}
            </div>
          {/each}
        </div>
        <div class="set" aria-hidden="true">
          {#each R1 as n, i}
            <div class="chip">
              {#if IC[n]}
                <svg viewBox="0 0 24 24" width="22" height="22" fill="currentColor" aria-hidden="true"><path d={IC[n]}/></svg>
              {:else}
                <b style="background: {getChipBackground(i)}">{n[0].toUpperCase()}</b>
              {/if}
              {n}
            </div>
          {/each}
        </div>
      </div>
    </div>
    <div class="mq rev" id="mq2">
      <div class="tr">
        <div class="set">
          {#each R2 as n, i}
            <div class="chip">
              <b style="background: {getChipBackground(i)}">{n[0].toUpperCase()}</b>
              {n}
            </div>
          {/each}
        </div>
        <div class="set" aria-hidden="true">
          {#each R2 as n, i}
            <div class="chip">
              <b style="background: {getChipBackground(i)}">{n[0].toUpperCase()}</b>
              {n}
            </div>
          {/each}
        </div>
      </div>
    </div>
  </div>

  <section id="features">
    <div class="hd">
      <h2>Everything You Need to Talk to Your Terminal</h2>
      <p>A small, fast desktop app built for real daily use, not just a demo.</p>
    </div>
    <div class="grid">
      <div class="card hot" style="--i: 0">
        <div class="fv">
          <div class="kc"><span>Ctrl</span><span>Shift</span><span>V</span></div>
        </div>
        <h3>Global hotkey</h3>
        <p>Hold to talk or toggle. Rebindable.</p>
      </div>
      <div class="card" style="--i: 1">
        <div class="fv">
          <svg viewBox="0 0 120 64" aria-hidden="true">
            <circle class="ring" cx="60" cy="32" r="28" fill="none" stroke="#8f83ff" stroke-width="1.5" stroke-dasharray="4 6" opacity=".7"/>
            <rect x="40" y="12" width="40" height="40" rx="8" fill="#14143a" stroke="#8f83ff" stroke-width="2"/>
            <path d="M48 8v4M60 8v4M72 8v4M48 52v4M60 52v4M72 52v4M36 22h4M36 32h4M36 42h4M80 22h4M80 32h4M80 42h4" stroke="#8f83ff" stroke-width="2" stroke-linecap="round"/>
            <g class="lock">
              <rect x="52" y="30" width="16" height="12" rx="2.5" fill="#fff"/>
              <path d="M55 30v-4a5 5 0 0 1 10 0v4" fill="none" stroke="#fff" stroke-width="2.2"/>
            </g>
          </svg>
        </div>
        <h3>On-device speech</h3>
        <p>Speech is transcribed on-device and audio is never uploaded.</p>
      </div>
      <div class="card" style="--i: 0">
        <div class="fv">
          <svg viewBox="0 0 120 64" aria-hidden="true">
            <rect x="8" y="6" width="104" height="52" rx="8" fill="#0b0b1c" stroke="rgba(255,255,255,.2)"/>
            <circle cx="18" cy="15" r="2.2" fill="#ff5f57"/>
            <circle cx="25" cy="15" r="2.2" fill="#febc2e"/>
            <circle cx="32" cy="15" r="2.2" fill="#28c840"/>
            <text x="16" y="35" fill="#8f83ff" font-size="9" font-family="monospace">&gt;</text>
            <rect class="ty" x="26" y="29" width="64" height="5" rx="2.5" fill="#d9d9f2"/>
            <rect class="cu" x="93" y="27" width="3" height="9" rx="1" fill="#8f83ff"/>
            <rect class="ty t2" x="16" y="42" width="48" height="5" rx="2.5" fill="#8d8dab"/>
          </svg>
        </div>
        <h3>Types anywhere</h3>
        <p>Types at your cursor in any window.</p>
      </div>
      <div class="card" style="--i: 1">
        <div class="fv">
          <svg viewBox="0 0 120 64" aria-hidden="true">
            <circle cx="60" cy="32" r="24" fill="none" stroke="rgba(255,255,255,.12)" stroke-width="5"/>
            <circle class="pg" cx="60" cy="32" r="24" fill="none" stroke="#8f83ff" stroke-width="5" stroke-linecap="round"/>
            <text x="60" y="37" text-anchor="middle" fill="#fff" font-size="14" font-weight="700" font-family="Inter,Arial,sans-serif">&lt;1s</text>
          </svg>
        </div>
        <h3>Sub-second speed</h3>
        <p>Text appears in under a second.</p>
      </div>
      <div class="card" style="--i: 0">
        <div class="fv">
          <svg viewBox="0 0 120 64" aria-hidden="true">
            <rect x="10" y="12" width="100" height="10" rx="5" fill="rgba(255,255,255,.08)"/>
            <rect class="hv" x="10" y="12" width="100" height="10" rx="5" fill="rgba(255,255,255,.3)"/>
            <rect x="10" y="40" width="100" height="10" rx="5" fill="rgba(255,255,255,.08)"/>
            <rect class="lw" x="10" y="40" width="22" height="10" rx="5" fill="#8f83ff"/>
            <text x="38" y="48" fill="#fff" font-size="8" font-weight="600" font-family="Inter,Arial,sans-serif">&lt; 50 MB</text>
          </svg>
        </div>
        <h3>Featherweight</h3>
        <p>Idle memory under 50 MB.</p>
      </div>
      <div class="card" style="--i: 1">
        <div class="fv">
          <div class="vd" aria-hidden="true">
            <i class="s"></i><i class="s"></i>
            {#each Array(6) as _, i}
              <i class="v" style="animation-delay: {0.14 + i * 0.07}s"></i>
            {/each}
            <i class="s"></i><i class="s"></i><i class="s"></i>
            {#each Array(3) as _, i}
              <i class="v" style="animation-delay: {0.77 + i * 0.07}s"></i>
            {/each}
            <i class="s"></i><i class="s"></i><i class="s"></i>
          </div>
        </div>
        <h3>Voice activity detection</h3>
        <p>Silence ignored. Only your words typed.</p>
      </div>
    </div>
  </section>

  <section id="how" style="padding-top:20px">
    <div class="hd">
      <h2>Three Steps. No New Workflow.</h2>
      <p>Cordė sits in your system tray until you call it.</p>
    </div>
    <div class="bento">
      <div class="tile t2">
        <h3>1 · Press the shortcut</h3>
        <p>Hold to talk or tap to toggle. A subtle tray indicator shows when it's listening.</p>
        <div class="vis"><div class="orb"></div></div>
      </div>
      <div class="tile t2">
        <h3>2 · Speak naturally</h3>
        <p>A local Whisper model transcribes as you talk.</p>
        <div class="vis">
          <div class="eq">
            {#each eqBars as height, i}
              <i style="height: {height}px; animation-delay: {i * 0.07}s"></i>
            {/each}
          </div>
        </div>
      </div>
      <div class="tile t2">
        <h3>3 · It types for you</h3>
        <p>Your words land at the cursor, ready to send to your agent.</p>
        <div class="vis"><div class="mini"><b>&gt;</b> refactor the auth middleware…<br><em>✓</em> typed by Cordė</div></div>
      </div>
    </div>
  </section>

  <section id="teams" style="padding-top:20px">
    <div class="hd">
      <h2>Designed for Developers Who Live in the CLI</h2>
      <p>From long agent prompts to quick commands, speaking is faster than typing.</p>
    </div>
    <div class="bento">
      <div class="tile t3">
        <h3>Agent prompting</h3>
        <p>Dictate long, detailed instructions to Claude Code and other CLI agents without breaking flow.</p>
        <div class="vis"><div class="mini"><b>&gt;</b> add per-user rate limiting and write tests for every edge case<span class="cur"></span></div></div>
      </div>
      <div class="tile t3">
        <h3>Private by design</h3>
        <p>Speech is transcribed locally and audio isn't uploaded. Account, subscription, and license checks need internet access.</p>
        <div class="vis">
          <div class="chips">
            <span>Local speech</span><span>No audio uploads</span><span>No API key</span><span>Internet for license</span>
          </div>
        </div>
      </div>
    </div>
  </section>

  <section id="pricing">
    <div class="hd">
      <h2>Simple Pricing</h2>
      <p>Confirmed prices. Payments and account features are not available yet.</p>
    </div>
    <div class="price">
      <div class="plan pop">
        <small class="plan-label">FREE TRIAL</small>
        <h3>Try Cordė</h3>
        <strong class="plan-price">7 days</strong>
        <p class="plan-detail">Free</p>
        <span class="trial-tag">No credit card required</span>
      </div>
      <div class="plan">
        <small class="plan-label">MONTHLY</small>
        <h3>Monthly</h3>
        <strong class="plan-price">$2.99</strong>
        <p class="plan-detail">Per month after the free trial.</p>
      </div>
      <div class="plan">
        <small class="plan-label">ANNUAL</small>
        <h3>Annual</h3>
        <strong class="plan-price">$34.99</strong>
        <p class="plan-detail">Per year after the free trial.</p>
      </div>
    </div>
    <p class="pricing-status">All plans include unlimited local dictation on macOS, Windows, and Linux. One active device per account; device deactivation and transfer are planned.</p>
    <p class="legal-note">Payment, account, and license features are not available yet. Billing, device, and license terms require owner/legal review.</p>
  </section>

  <section id="faq" style="padding-top:20px">
    <div class="hd"><h2>Questions</h2></div>
    <div class="faq">
      <details><summary>Does my voice leave my computer?</summary><p>Speech recognition runs locally, and audio isn't uploaded for transcription. Account, subscription, and license checks require an internet connection.</p></details>
      <details><summary>Which apps does it work with?</summary><p>Any app that accepts keyboard input. It was designed for terminals and CLI agents like Claude Code.</p></details>
      <details><summary>How accurate is it?</summary><p>It uses OpenAI's Whisper models in an optimized runtime. Smaller models are faster, and you can pick a larger one for more accuracy.</p></details>
      <details><summary>How much does Cordė cost?</summary><p>Cordė includes a 7-day free trial with no credit card required. After the trial, prices are $2.99/month or $34.99/year. Payments aren't available yet. Once subscriptions launch, an active plan will be required to dictate; the planned upgrade prompt appears when you try to dictate, not on every app launch. Existing transcripts and settings won't be deleted or locked.</p></details>
      <details><summary>Can I use Cordė on a replacement device?</summary><p>The plan allows one active device per account. A way to deactivate or transfer activation when replacing a device is planned; account and license features aren't available yet.</p></details>
      <details>
        <summary>How do I open Cordė on macOS if I see a security popup?</summary>
        <p>
          <strong>First-time launch on macOS:</strong><br/>
          1. Open the <code>.dmg</code> and drag <strong>Cordė</strong> into <strong>Applications</strong>.<br/>
          2. <strong>Right-click</strong> <strong>Cordė</strong> in Applications and select <strong>Open</strong> <em>(or on macOS Sequoia, go to <strong>System Settings → Privacy &amp; Security</strong> and click <strong>Open Anyway</strong>)</em>.<br/>
          3. If macOS says the app cannot be opened, run <code>xattr -cr /Applications/Cordė.app</code> in Terminal. You only need to do this once!
        </p>
      </details>
      <details>
        <summary>How do I install Cordė on Windows if SmartScreen shows a popup?</summary>
        <p>
          <strong>First-time launch on Windows:</strong><br/>
          1. Open the downloaded <code>Corde-setup.exe</code> installer.<br/>
          2. If Windows Defender SmartScreen shows <em>“Windows protected your PC”</em>, click <strong>More info</strong>.<br/>
          3. Click <strong>Run anyway</strong> to complete the installation. You only need to do this once!
        </p>
      </details>
      <details><summary>How do I say the name?</summary><p>“Cord-ay”.</p></details>
    </div>
  </section>

  <section style="padding-top:20px">
    <div class="final">
      <h2>Stop typing. Start speaking.</h2>
      <a class="btn lg" href="#pricing">View pricing</a>
    </div>
  </section>

  <footer>
    <div>© Cordė · Voice to terminal</div>
    <div>Code at the speed of thought.</div>
    <a class="footer-release-link" href="https://github.com/corde-voice/corde-releases/releases/latest" target="_blank" rel="noopener noreferrer">
      Download Cordė for macOS &amp; Windows
    </a>
    <a class="footer-release-link" href="https://x.com/Corde_dex" target="_blank" rel="noopener noreferrer">
      Follow on X @Corde_dex
    </a>
  </footer>

  {#if downloadGuideModal}
    <div
      class="dl-modal-backdrop"
      role="dialog"
      aria-modal="true"
      aria-labelledby="dl-modal-title"
      tabindex="-1"
      onclick={(e) => { if (e.target === e.currentTarget) closeDownloadGuide(); }}
      onkeydown={(e) => { if (e.key === 'Escape') closeDownloadGuide(); }}
    >
      <div class="dl-modal-card">
        <button class="dl-modal-close" type="button" aria-label="Close instructions" onclick={closeDownloadGuide}>✕</button>
        {#if downloadGuideModal === 'mac'}
          <p class="dl-modal-eyebrow">DOWNLOADING CORDĖ FOR MACOS</p>
          <h3 id="dl-modal-title">First-time launch on macOS:</h3>
          <ol class="dl-modal-steps">
            <li>Open the <code>.dmg</code> and drag <strong>Cordė</strong> into <strong>Applications</strong>.</li>
            <li><strong>Right-click</strong> <strong>Cordė</strong> in Applications and select <strong>Open</strong> <em>(or on macOS Sequoia, go to <strong>System Settings → Privacy &amp; Security</strong> and click <strong>Open Anyway</strong>)</em>.</li>
            <li>
              If macOS says the app cannot be opened, run <code>xattr -cr /Applications/Cordė.app</code> in Terminal. You only need to do this once!
              <div class="dl-modal-cmd-row">
                <code>xattr -cr /Applications/Cordė.app</code>
                <button type="button" class="dl-modal-copy-btn" onclick={copyMacTerminalCommand}>
                  {copiedXattrCommand ? "Copied!" : "Copy"}
                </button>
              </div>
            </li>
          </ol>
          <p class="dl-modal-alt">
            Using an Intel Mac? <a href="https://github.com/corde-voice/corde-releases/releases/latest/download/Corde-macos-x64.dmg">Download Intel (.dmg)</a>
          </p>
        {:else}
          <p class="dl-modal-eyebrow">DOWNLOADING CORDĖ FOR WINDOWS</p>
          <h3 id="dl-modal-title">First-time launch on Windows:</h3>
          <ol class="dl-modal-steps">
            <li>Open the downloaded <code>Corde-setup.exe</code> installer.</li>
            <li>If Windows Defender SmartScreen shows <em>“Windows protected your PC”</em>, click <strong>More info</strong>.</li>
            <li>Click <strong>Run anyway</strong> to complete the installation. You only need to do this once!</li>
          </ol>
        {/if}
        <button type="button" class="btn dl-modal-done" onclick={closeDownloadGuide}>Got it</button>
      </div>
    </div>
  {/if}
  {/if}
</div>

<style>
  /* Local overrides if needed, but most is in app.css */
  .billing-success {
    display: grid;
    min-height: 100vh;
    place-items: center;
    padding: 48px 20px;
  }

  .billing-success-card {
    width: min(100%, 440px);
    padding: clamp(24px, 5vw, 36px);
    border: 1px solid rgba(255, 255, 255, 0.12);
    border-radius: 20px;
    background: rgba(13, 13, 32, 0.84);
    box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  }

  .billing-success-icon {
    display: grid;
    width: 42px;
    height: 42px;
    margin-bottom: 18px;
    place-items: center;
    border-radius: 50%;
    background: rgba(143, 131, 255, 0.16);
    color: #b9b0ff;
    font-size: 23px;
  }

  .billing-success-eyebrow {
    margin: 0 0 8px;
    color: #b9b0ff;
    font-size: 12px;
    font-weight: 700;
    letter-spacing: 0.12em;
  }

  .billing-success h1 {
    margin: 0;
    font-size: clamp(32px, 7vw, 42px);
  }

  .billing-success-copy,
  .billing-success-note {
    color: rgba(255, 255, 255, 0.72);
    line-height: 1.55;
  }

  .billing-success-copy {
    margin: 12px 0 20px;
  }

  .billing-success-note {
    margin: 18px 0 0;
    font-size: 13px;
  }

  .billing-success-home {
    display: inline-block;
    margin-top: 18px;
    color: rgba(255, 255, 255, 0.72);
    font-size: 13px;
    text-decoration: underline;
    text-underline-offset: 3px;
  }

  .keys {
    display: flex;
    gap: 6px;
  }

  .dl-modal-backdrop {
    position: fixed;
    inset: 0;
    z-index: 999;
    display: grid;
    place-items: center;
    padding: 20px;
    background: rgba(5, 6, 15, 0.78);
    backdrop-filter: blur(8px);
  }

  .dl-modal-card {
    position: relative;
    width: min(100%, 490px);
    padding: clamp(24px, 4vw, 34px);
    border: 1px solid rgba(143, 131, 255, 0.28);
    border-radius: 20px;
    background: rgba(14, 15, 36, 0.96);
    box-shadow: 0 24px 70px rgba(0, 0, 0, 0.55);
    text-align: left;
  }

  .dl-modal-close {
    position: absolute;
    top: 16px;
    right: 16px;
    width: 32px;
    height: 32px;
    border: 1px solid rgba(255, 255, 255, 0.12);
    border-radius: 8px;
    background: rgba(255, 255, 255, 0.05);
    color: rgba(255, 255, 255, 0.7);
    cursor: pointer;
    font-size: 14px;
  }

  .dl-modal-close:hover {
    background: rgba(255, 255, 255, 0.12);
    color: #fff;
  }

  .dl-modal-eyebrow {
    margin: 0 0 6px;
    color: #b9b0ff;
    font-size: 11px;
    font-weight: 700;
    letter-spacing: 0.12em;
  }

  .dl-modal-card h3 {
    margin: 0 0 16px;
    font-size: 22px;
    color: #fff;
  }

  .dl-modal-steps {
    margin: 0 0 18px;
    padding-left: 20px;
    color: rgba(255, 255, 255, 0.84);
    font-size: 14px;
    line-height: 1.6;
  }

  .dl-modal-steps li {
    margin-bottom: 10px;
  }

  .dl-modal-steps code {
    padding: 2px 6px;
    border-radius: 6px;
    background: rgba(143, 131, 255, 0.16);
    color: #d9d4ff;
    font-size: 12.5px;
    font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
  }

  .dl-modal-cmd-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    margin-top: 8px;
    padding: 8px 12px;
    border: 1px solid rgba(255, 255, 255, 0.12);
    border-radius: 10px;
    background: rgba(6, 7, 18, 0.85);
  }

  .dl-modal-cmd-row code {
    background: transparent;
    padding: 0;
    color: #e6e2ff;
  }

  .dl-modal-copy-btn {
    padding: 5px 11px;
    border: 1px solid rgba(143, 131, 255, 0.35);
    border-radius: 7px;
    background: rgba(143, 131, 255, 0.18);
    color: #fff;
    font-size: 12px;
    font-weight: 600;
    cursor: pointer;
    white-space: nowrap;
  }

  .dl-modal-copy-btn:hover {
    background: rgba(143, 131, 255, 0.32);
  }

  .dl-modal-alt {
    margin: 0 0 18px;
    color: rgba(255, 255, 255, 0.62);
    font-size: 13px;
  }

  .dl-modal-alt a {
    color: #b9b0ff;
    text-decoration: underline;
    text-underline-offset: 3px;
  }

  .dl-modal-done {
    width: 100%;
    justify-content: center;
    cursor: pointer;
  }
</style>
