/**
 * @license
 * SPDX-License-Identifier: Apache-2.0
 */

import { useState, useEffect, useRef, ChangeEvent, FormEvent, ReactNode } from "react";
import { Search, User, Tv, Calendar, Home, Play, PlayCircle, Pause, Radio, Info, Sun, Moon, Maximize, Settings, Volume2, VolumeX, CheckCircle2, Shield, LogOut, LogIn, Heart, X, Lock, Terminal, Zap, Clock, History, MousePointer2, Sliders, ChevronLeft, ChevronRight, Mic, Layers, Filter, Sparkles, Camera, Palette, Layout, MessageSquare, Eye, EyeOff, ExternalLink } from "lucide-react";
import Hls from "hls.js";
import { motion, AnimatePresence, MotionConfig } from "motion/react";
import { auth, db } from "./firebase";
import { signInWithEmailAndPassword, createUserWithEmailAndPassword, signOut, onAuthStateChanged, updateProfile, sendPasswordResetEmail, User as FirebaseUser, GoogleAuthProvider, signInWithPopup } from "firebase/auth";
import { doc, getDoc, setDoc, collection, getDocs, serverTimestamp, updateDoc, arrayUnion, getDocFromServer, onSnapshot } from "firebase/firestore";

import { channels, Channel } from "./channels";

const PALETTES = [
  { name: "Purple", color: "#a855f7" },
  { name: "Blue", color: "#3b82f6" },
  { name: "Indigo", color: "#6366f1" },
  { name: "Rose", color: "#f43f5e" },
  { name: "Green", color: "#10b981" },
  { name: "Amber", color: "#f59e0b" },
  { name: "Slate", color: "#64748b" },
];

// Test connection as per critical directive
async function testFirestoreConnection() {
  try {
    await getDocFromServer(doc(db, 'test', 'connection'));
  } catch (error) {
    if (error instanceof Error && error.message.includes('the client is offline')) {
      console.error("Please check your Firebase configuration. The client is offline.");
    }
  }
}
testFirestoreConnection();

const SettingsIcon = ({ className }: { className?: string }) => (
  <Settings className={`${className} flex-shrink-0`} />
);

const SplashScreen = ({ isDark }: { isDark: boolean }) => (
  <motion.div
    initial={{ opacity: 1 }}
    exit={{ opacity: 0 }}
    transition={{ duration: 0.8 }}
    className={`fixed inset-0 z-[100] flex flex-col items-center justify-center ${
      isDark 
        ? "bg-gradient-to-br from-rose-950 via-primary/20 to-red-950" 
        : "bg-gradient-to-br from-rose-200 via-primary/10 to-red-100"
    }`}
  >
    <motion.div
      initial={{ scale: 0.8, opacity: 0 }}
      animate={{ scale: 1, opacity: 1 }}
      transition={{ duration: 0.5, ease: "easeOut" }}
      className="flex flex-col items-center space-y-12"
    >
      <div className="relative">
        <motion.img 
          initial={{ scale: 0.9 }}
          animate={{ scale: [0.9, 1.05, 0.9] }}
          transition={{ duration: 4, repeat: Infinity, ease: "easeInOut" }}
          src="https://static.wikia.nocookie.net/ftv/images/9/93/Vpl.png/revision/latest?cb=20260412135144&path-prefix=vi" 
          alt="Vplay Logo" 
          className="h-56 w-56 md:h-72 md:w-72 object-contain drop-shadow-[0_0_30px_rgba(168,85,247,0.5)]"
          referrerPolicy="no-referrer"
        />
      </div>
    </motion.div>
  </motion.div>
);

const Sparkles2 = ({ className }: { className?: string }) => (
  <svg 
    viewBox="0 0 24 24" 
    fill="none" 
    stroke="currentColor" 
    strokeWidth="2" 
    strokeLinecap="round" 
    strokeLinejoin="round" 
    className={className}
  >
    <path d="m12 3-1.912 5.813a2 2 0 0 1-1.275 1.275L3 12l5.813 1.912a2 2 0 0 1 1.275 1.275L12 21l1.912-5.813a2 2 0 0 1 1.275-1.275L21 12l-5.813-1.912a2 2 0 0 1-1.275-1.275L12 3Z" />
    <circle cx="19" cy="5" r="2" fill="currentColor" stroke="none" />
  </svg>
);

const baseTabs = [
  { name: "Trang chủ", icon: Home, id: "Trang chủ" },
  { name: "Phát sóng", icon: Tv, id: "Phát sóng" },
  { name: "Phát lại", icon: History, id: "Phát lại" },
  { name: "Lịch phát sóng", icon: Calendar, id: "Lịch phát sóng" },
  { name: "Sự kiện", icon: Sparkles, id: "Sự kiện" },
  { name: "Video", icon: Play, id: "Video" },
  { name: "Cài đặt", icon: Settings, id: "Cài đặt" },
];

// Channel type is imported from channels.ts

const handleFirestoreError = (error: any, operationType: string, path: string) => {
  const errInfo = {
    error: error?.message || String(error),
    code: error?.code,
    operationType,
    path,
    authInfo: {
      userId: auth.currentUser?.uid,
      email: auth.currentUser?.email,
      emailVerified: auth.currentUser?.emailVerified
    }
  };
  console.error("Firestore Error Details:", JSON.stringify(errInfo, null, 2));
  throw new Error(`Firestore operation failed: ${errInfo.error}`);
};

function LiquidModal({ isOpen, onClose, children, isDark, title, description, liquidGlass }: { 
  isOpen: boolean, 
  onClose: () => void, 
  children?: ReactNode, 
  isDark: boolean,
  title?: string,
  description?: string,
  liquidGlass: boolean
}) {
  return (
    <AnimatePresence>
      {isOpen && (
        <div className="fixed inset-0 z-[200] flex items-center justify-center p-4">
          <motion.div
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            onClick={onClose}
            className={`absolute inset-0 bg-black/40 ${liquidGlass ? "backdrop-blur-sm" : ""}`}
          />
          <motion.div
            initial={{ scale: 0.8, opacity: 0, y: 20 }}
            animate={{ scale: 1, opacity: 1, y: 0 }}
            exit={{ scale: 0.8, opacity: 0, y: 20 }}
            transition={{ type: "spring", damping: 25, stiffness: 300 }}
            className={`relative w-full max-w-sm overflow-hidden border shadow-2xl ${
              isDark 
                ? "bg-slate-900/90 border-white/10 text-white" 
                : "bg-white/90 border-white/60 text-slate-900"
            } ${
              liquidGlass ? "rounded-[40px] backdrop-blur-3xl" : "rounded-2xl backdrop-blur-none"
            }`}
          >
            <div className="p-8 text-center">
              {title && <h3 className={`text-2xl font-bold mb-2 ${isDark ? "text-white" : "text-slate-900"}`}>{title}</h3>}
              {description && <p className={`${isDark ? "text-white/60" : "text-black/60"} text-sm leading-relaxed mb-6 font-medium`}>{description}</p>}
              {children}
            </div>
          </motion.div>
        </div>
      )}
    </AnimatePresence>
  );
}

function Tooltip({ text, show, targetRect }: { text: string, show: boolean, targetRect: DOMRect | null }) {
  return (
    <AnimatePresence>
      {show && targetRect && (
        <motion.div
          initial={{ opacity: 0, y: 10, scale: 0.8 }}
          animate={{ opacity: 1, y: 0, scale: 1 }}
          exit={{ opacity: 0, y: 10, scale: 0.8 }}
          style={{ 
            position: 'fixed', 
            top: targetRect.top - 50, 
            left: targetRect.left + (targetRect.width / 2),
            translateX: '-50%'
          }}
          className="px-4 py-2 bg-white/80 backdrop-blur-xl text-slate-900 text-[12px] font-black rounded-2xl whitespace-nowrap pointer-events-none z-[100] shadow-[0_10px_30px_rgba(0,0,0,0.1)] border border-white/40"
        >
          {text}
          <div className="absolute top-full left-1/2 -translate-x-1/2 border-[6px] border-transparent border-t-white/80" />
        </motion.div>
      )}
    </AnimatePresence>
  );
}

function ChannelLogo({ src, alt, className, isDark }: { src: string, alt: string, className?: string, isDark: boolean }) {
  const [error, setError] = useState(false);

  if (error || !src) {
    return (
      <div className={`${className} flex flex-col items-center justify-center bg-slate-800/50 rounded-lg border border-slate-700/50 p-1 text-center`}>
        <Tv className="h-6 w-6 mb-1 text-slate-500" />
        <span className="text-[10px] font-bold leading-tight line-clamp-2 uppercase opacity-60">{alt}</span>
      </div>
    );
  }

  const scaleMap: { [key: string]: string } = {
    "Lâm Đồng 1 (LTV1)": "md:scale-[1.4]",
    "LĐA1": "md:scale-[1.4]",
    "Đà Nẵng 1 (DNRT1)": "scale-[1.5] md:scale-[1.7]",
    "Đà Nẵng 2 (DNRT2)": "scale-[1.4] md:scale-[1.7]",
    "Thái Nguyên (TN)": "md:scale-[1.5]",
    "Điện Biên (ĐTV)": "md:scale-[0.8]",
    "Hưng Yên (HYTV)": "md:scale-[1.7]",
    "Đồng Tháp 1 (THĐT1)": "scale-[2.0] md:scale-[1.4]",
    "Huế (HueTV)": "md:scale-[1.4]",
    "Tây Ninh (TN)": "md:scale-[1.4]",
    "H1": "md:scale-[1.4]",
    "H2": "md:scale-[1.4]",
    "Đắk Lắk (DRT)": "scale-[1.2] md:scale-[1.4]",
    "ĐNNRTV1": "scale-[1.1] md:scale-[1.1]",
    "ĐNNRTV2": "scale-[1.1] md:scale-[1.1]",
    "Nghệ An (NTV)": "md:scale-[1.4]",
    "Quảng Ngãi 1 (QNgTV1)": "md:scale-[1.5]",
    "Quảng Ngãi 2 (QNgTV2)": "md:scale-[1.5]",
    "HTV Thể Thao": "scale-[1.5] md:scale-[1.5]",
    "VTV1": "scale-[1.14] md:scale-[0.92]",
    "VTV7": "scale-[1.24] md:scale-[1.01]",
    "VTV10": "scale-[1.11] md:scale-[1.0]"
  };

  const scaleClass = scaleMap[alt] || (alt.startsWith("VTV") ? "md:scale-[0.9]" : "");

  return (
    <img 
      src={src} 
      alt={alt} 
      referrerPolicy="no-referrer"
      onError={() => setError(true)}
      className={`${className} object-contain transition-all duration-300 ${!isDark ? "drop-shadow-[0_8px_16px_rgba(0,0,0,0.4)]" : ""} ${scaleClass}`} 
    />
  );
}

function ChannelCard({ ch, onClick, isDark, isActive, favorites, toggleFavorite, liquidGlass }: {
  ch: Channel,
  onClick: () => void,
  isDark: boolean,
  isActive?: boolean,
  favorites: string[],
  toggleFavorite: (ch: Channel) => void,
  liquidGlass: boolean,
  key?: string | number
}) {
  const isMaintenance = ch.status === "maintenance";
  const isMovie = ch.status === "movie";

  return (
    <div className="relative group">
      <motion.button
        whileHover={{ scale: 1.12, boxShadow: isActive ? "0 0 40px rgba(168,85,247,0.7)" : "0 0 25px rgba(0,0,0,0.15)" }}
        whileTap={{ scale: 0.95, rotate: -1 }}
        transition={{ type: "spring", stiffness: 400, damping: 25 }}
        onClick={onClick}
        className={`w-full aspect-video p-4 md:p-8 flex items-center justify-center transition-all duration-200 border relative overflow-hidden ${
          liquidGlass ? "rounded-2xl backdrop-blur-2xl border-white/20 shadow-2xl" : "rounded-lg backdrop-blur-none border-slate-200"
        } ${
          isActive
            ? `bg-white/20 border-primary ring-2 ring-primary/50 shadow-[0_0_20px_rgba(168,85,247,0.4)]`
            : isDark
            ? "bg-white/5 border-white/5 shadow-sm shadow-black/20"
            : "bg-white/10 border-white/20 shadow-[0_8px_30px_rgb(0,0,0,0.05)]"
        }`}
      >
        {/* Status indicators */}
        <div className="absolute top-2.5 left-2.5 flex items-center gap-1.5 z-20">
          <div className={`h-2 w-2 rounded-full shadow-[0_0_8px_rgba(0,0,0,0.2)] ${
            isMaintenance ? "bg-amber-500 shadow-amber-500/50" : 
            isMovie ? "bg-blue-400 shadow-blue-400/50" : 
            "bg-green-500 shadow-green-500/50 animate-pulse"
          }`} />
          <span className={`text-[8px] font-black uppercase tracking-tighter opacity-0 group-hover:opacity-100 transition-opacity duration-300 ${
            isMaintenance ? "text-amber-500" : isMovie ? "text-blue-400" : "text-green-500"
          }`}>
            {isMaintenance ? (ch.statusLabel || "Trực tiếp") : isMovie ? "Video" : "Bình thường"}
          </span>
        </div>

        {/* Hover reveal name */}
        <div className="absolute inset-x-0 bottom-0 py-2.5 bg-gradient-to-t from-black/80 to-transparent opacity-0 group-hover:opacity-100 transition-opacity duration-300 z-10 flex items-center justify-center px-4">
          <span className="text-white text-[9px] font-bold text-center line-clamp-1 drop-shadow-md">
            {ch.name}
          </span>
        </div>

        <ChannelLogo src={ch.logo} alt={ch.name} className={`w-full h-full transition-all duration-300 group-hover:scale-95 group-hover:opacity-40 ${isMaintenance ? "grayscale opacity-40" : ""}`} isDark={isDark} />
      </motion.button>
      <button 
        onClick={(e) => { e.stopPropagation(); toggleFavorite(ch); }}
        className={`absolute top-2 right-2 p-1.5 rounded-full bg-black/40 opacity-0 group-hover:opacity-100 transition-opacity hover:bg-black/60 z-30 ${favorites.includes(ch.name) ? "text-red-500" : "text-white"}`}
      >
        <Heart className={`h-4 w-4 ${favorites.includes(ch.name) ? "fill-red-500" : ""}`} />
      </button>
    </div>
  );
}

function HomeContent({ setActiveTab, setActiveChannel, isDark, favorites, toggleFavorite, liquidGlass }: {
  setActiveTab: (tab: string) => void,
  setActiveChannel: (ch: typeof channels[0]) => void,
  isDark: boolean,
  favorites: string[],
  toggleFavorite: (ch: typeof channels[0]) => void,
  liquidGlass: boolean
}) {
  const [slideIndex, setSlideIndex] = useState(0);
  const [direction, setDirection] = useState(0);
  const [randomChannels, setRandomChannels] = useState<typeof channels>([]);
  const slides = [
    "https://plain-apac-prod-public.komododecks.com/202604/06/0rdrbV8FYCssv6LnT4aJ/image.png",
    "https://plain-apac-prod-public.komododecks.com/202604/06/DN6JPkubjkRfKgJlYYIa/image.png"
  ];

  const paginate = (newDirection: number) => {
    setDirection(newDirection);
    setSlideIndex((prev) => (prev + newDirection + slides.length) % slides.length);
  };

  useEffect(() => {
    const shuffled = [...channels].sort(() => 0.5 - Math.random());
    setRandomChannels(shuffled.slice(0, 12));

    const interval = setInterval(() => {
      paginate(1);
    }, 3000);
    return () => clearInterval(interval);
  }, []);

  const variants = {
    enter: (direction: number) => ({
      x: direction > 0 ? '100%' : '-100%',
      opacity: 0
    }),
    center: {
      zIndex: 1,
      x: 0,
      opacity: 1
    },
    exit: (direction: number) => ({
      zIndex: 0,
      x: direction < 0 ? '100%' : '-100%',
      opacity: 0
    })
  };

  const favoriteChannels = channels.filter(ch => favorites.includes(ch.name));

  return (
    <div className="p-4 md:p-6 space-y-10">
      {/* Welcome Message */}
      <div className="text-center py-6 space-y-6">
        <motion.div
          initial={{ scale: 0.5, opacity: 0 }}
          animate={{ scale: 1, opacity: 1 }}
          className="flex justify-center"
        >
          <img 
            src="https://static.wikia.nocookie.net/ftv/images/9/93/Vpl.png/revision/latest?cb=20260412135144&path-prefix=vi" 
            alt="Vplay Logo" 
            className="h-40 w-40 md:h-56 md:w-56 object-contain drop-shadow-2xl"
            referrerPolicy="no-referrer"
          />
        </motion.div>
        <div>
          <motion.h1 
            initial={{ opacity: 0, y: -20 }}
            animate={{ opacity: 1, y: 0 }}
            className={`text-3xl md:text-5xl font-black tracking-tighter ${isDark ? "text-white" : "text-slate-900"}`}
          >
            Chào mừng đến với <span className="text-primary">Vplay!</span>
          </motion.h1>
          <p className={`mt-2 text-sm md:text-base font-medium ${isDark ? "text-slate-400" : "text-slate-600"}`}>
            Trải nghiệm truyền hình trực tuyến đỉnh cao
          </p>
        </div>
      </div>

      {/* Slider */}
      <div className={`relative w-full max-w-5xl mx-auto aspect-[2.4/1] overflow-hidden shadow-2xl border border-white/10 bg-slate-900/20 group ${
        liquidGlass ? "rounded-3xl" : "rounded-xl"
      }`}>
        <AnimatePresence initial={false} custom={direction}>
          <motion.img
            key={slideIndex}
            src={slides[slideIndex]}
            custom={direction}
            variants={variants}
            initial="enter"
            animate="center"
            exit="exit"
            transition={{
              x: { type: "spring", stiffness: 300, damping: 30 },
              opacity: { duration: 0.2 }
            }}
            referrerPolicy="no-referrer"
            className="absolute inset-0 w-full h-full object-cover"
          />
        </AnimatePresence>
        <div className="absolute inset-0 bg-gradient-to-t from-black/60 via-transparent to-transparent pointer-events-none z-10" />
        
        {/* Navigation Arrows */}
        <button 
          onClick={() => paginate(-1)}
          className={`absolute left-4 top-1/2 -translate-y-1/2 z-20 p-2 rounded-full bg-black/20 hover:bg-black/40 text-white transition-all opacity-0 group-hover:opacity-100 ${
            liquidGlass ? "backdrop-blur-md" : "backdrop-blur-none"
          }`}
        >
          <ChevronLeft className="h-6 w-6" />
        </button>
        <button 
          onClick={() => paginate(1)}
          className={`absolute right-4 top-1/2 -translate-y-1/2 z-20 p-2 rounded-full bg-black/20 hover:bg-black/40 text-white transition-all opacity-0 group-hover:opacity-100 ${
            liquidGlass ? "backdrop-blur-md" : "backdrop-blur-none"
          }`}
        >
          <ChevronRight className="h-6 w-6" />
        </button>

        {/* Slide Indicators */}
        <div className="absolute bottom-4 left-1/2 -translate-x-1/2 flex gap-2 z-20">
          {slides.map((_, i) => (
            <button 
              key={i} 
              onClick={() => {
                setDirection(i > slideIndex ? 1 : -1);
                setSlideIndex(i);
              }}
              className={`h-1.5 rounded-full transition-all duration-500 ${i === slideIndex ? "w-8 bg-primary" : "w-2 bg-white/30 hover:bg-white/50"}`}
            />
          ))}
        </div>
      </div>

      {/* Favorites Section */}
      {favoriteChannels.length > 0 && (
        <div className="space-y-4">
          <div className="flex items-center gap-2">
            <Heart className="h-5 w-5 text-red-500 fill-red-500" />
            <h3 className={`text-xl font-bold ${isDark ? "text-white" : "text-slate-950"}`}>Kênh yêu thích</h3>
          </div>
          <div className="grid grid-cols-3 sm:grid-cols-3 md:grid-cols-6 gap-3 md:gap-4">
            {favoriteChannels.map(ch => (
              <ChannelCard 
                key={ch.name} 
                ch={ch} 
                onClick={() => setActiveChannel(ch)} 
                isDark={isDark} 
                favorites={favorites} 
                toggleFavorite={toggleFavorite} 
                liquidGlass={liquidGlass}
              />
            ))}
          </div>
        </div>
      )}

      {/* Suggested Channels */}
      <div className="space-y-4">
        <h3 className={`text-xl font-bold ${isDark ? "text-white" : "text-slate-950"}`}>Kênh đề xuất</h3>
        <div className="grid grid-cols-3 sm:grid-cols-3 md:grid-cols-6 gap-3 md:gap-4">
          {randomChannels.map(ch => (
            <ChannelCard 
              key={`${ch.name}-${ch.stream}`} 
              ch={ch} 
              onClick={() => setActiveChannel(ch)} 
              isDark={isDark} 
              favorites={favorites} 
              toggleFavorite={toggleFavorite} 
              liquidGlass={liquidGlass}
            />
          ))}
        </div>
      </div>
    </div>
  );
}

function TVContent({ active, setActive, isDark, favorites, toggleFavorite, user, onLogin, isDev, liquidGlass, sortOrder, setSortOrder }: { 
  active: Channel, 
  setActive: (ch: Channel) => void, 
  isDark: boolean,
  favorites: string[],
  toggleFavorite: (ch: Channel) => void,
  user: any,
  onLogin: () => void,
  isDev?: boolean,
  liquidGlass: boolean,
  sortOrder: "default" | "az" | "za",
  setSortOrder: (val: "default" | "az" | "za") => void
}) {
  const videoRef = useRef<HTMLVideoElement>(null);
  const hlsRef = useRef<Hls | null>(null);
  const playPromiseRef = useRef<Promise<void> | null>(null);
  const [isPlaying, setIsPlaying] = useState(true);
  const [isMuted, setIsMuted] = useState(true);
  const [volume, setVolume] = useState(1);
  const [levels, setLevels] = useState<Hls.Level[]>([]);
  const [currentLevel, setCurrentLevel] = useState(-1);
  const [showQualityMenu, setShowQualityMenu] = useState(false);
  const [isBuffering, setIsBuffering] = useState(false);
  const [searchQuery, setSearchQuery] = useState("");
  const [filterType, setFilterType] = useState<string>("Tất cả");

  // categories definition removed to avoid duplication

  const [showSortMenu, setShowSortMenu] = useState(false);

  const filteredChannels = channels
    .filter(ch => {
      const matchesSearch = ch.name.toLowerCase().includes(searchQuery.toLowerCase());
      const matchesType = filterType === "Tất cả" 
        || (filterType === "Hoạt động" && ch.status !== "maintenance")
        || (filterType === "Trực tiếp" && ch.status === "maintenance")
        || (filterType === "Bảo trì" && ch.status === "maintenance")
        || ch.category === filterType;
      return matchesSearch && matchesType;
    })
    .sort((a, b) => {
      if (sortOrder === "default") return 0;
      if (sortOrder === "az") return a.name.localeCompare(b.name);
      return b.name.localeCompare(a.name);
    });

  const CATEGORY_ORDER = ["VTV", "HTV", "VTVcab", "Thiết yếu", "Phim & Video", "Địa phương", "Phát thanh"];
  const filteredCategories = CATEGORY_ORDER.filter(cat => 
    filteredChannels.some(ch => ch.category === cat)
  );

  useEffect(() => {
    if (!user && !isDev) return;
    const video = videoRef.current;
    if (!video) return;

    // Track watched channel
    if (user) {
      const userRef = doc(db, "users", user.uid);
      updateDoc(userRef, {
        watchedChannels: arrayUnion(active.name)
      }).catch(err => {
        try {
          handleFirestoreError(err, "update", `users/${user.uid}`);
        } catch (e) {
          console.error("Watched channels update failed:", e);
        }
      });
    }

    video.volume = volume;

    const safePlay = async () => {
      if (!video) return;
      try {
        playPromiseRef.current = video.play();
        await playPromiseRef.current;
      } catch (e: any) {
        if (e.name !== "AbortError") {
          throw e;
        }
      } finally {
        playPromiseRef.current = null;
      }
    };

    const handleWaiting = () => setIsBuffering(true);
    const handlePlaying = () => setIsBuffering(false);
    const handleCanPlay = () => setIsBuffering(false);

    video.addEventListener('waiting', handleWaiting);
    video.addEventListener('playing', handlePlaying);
    video.addEventListener('canplay', handleCanPlay);

    const isMp4 = active.stream.toLowerCase().endsWith('.mp4') || active.stream.includes('.mp4?');
    let hls: Hls | null = null;

    if (isMp4) {
      if (hlsRef.current) {
        hlsRef.current.destroy();
        hlsRef.current = null;
      }
      video.src = active.stream;
      video.addEventListener('loadedmetadata', () => {
        safePlay().catch(() => {
          video.muted = true;
          setIsMuted(true);
          safePlay();
        });
      });
    } else if (Hls.isSupported()) {
      if (hlsRef.current) {
        hlsRef.current.destroy();
      }
      hls = new Hls();
      hlsRef.current = hls;
      hls.loadSource(active.stream);
      hls.attachMedia(video);
      hls.on(Hls.Events.MANIFEST_PARSED, () => {
        safePlay().catch(e => {
          console.warn("Autoplay prevented, trying muted", e);
          video.muted = true;
          setIsMuted(true);
          safePlay();
        });
        setLevels(hls!.levels);
        setCurrentLevel(hls!.currentLevel);
      });
      hls.on(Hls.Events.LEVEL_SWITCHED, (_, data) => {
        setCurrentLevel(data.level);
      });
    } else if (video.canPlayType('application/vnd.apple.mpegurl')) {
      video.src = active.stream;
      video.addEventListener('loadedmetadata', () => {
        safePlay().catch(() => {
          video.muted = true;
          setIsMuted(true);
          safePlay();
        });
      });
    }

    return () => {
      if (video) {
        video.removeEventListener('waiting', handleWaiting);
        video.removeEventListener('playing', handlePlaying);
        video.removeEventListener('canplay', handleCanPlay);
      }
      if (hlsRef.current) {
        hlsRef.current.destroy();
        hlsRef.current = null;
      }
    };
  }, [active, user]);

  const togglePlay = async () => {
    const video = videoRef.current;
    if (!video) return;
    if (video.paused) {
      try {
        playPromiseRef.current = video.play();
        setIsPlaying(true);
        await playPromiseRef.current;
      } catch (e: any) {
        if (e.name !== "AbortError") {
          console.error("Playback failed:", e);
          setIsPlaying(false);
        }
      } finally {
        playPromiseRef.current = null;
      }
    } else {
      if (playPromiseRef.current) {
        try {
          await playPromiseRef.current;
        } catch (e) {
          // Ignore play errors
        }
      }
      video.pause();
      setIsPlaying(false);
    }
  };

  const toggleMute = () => {
    const video = videoRef.current;
    if (!video) return;
    video.muted = !video.muted;
    setIsMuted(video.muted);
  };

  const handleVolumeChange = (e: ChangeEvent<HTMLInputElement>) => {
    const val = parseFloat(e.target.value);
    setVolume(val);
    if (videoRef.current) {
      videoRef.current.volume = val;
      if (val > 0 && isMuted) {
        videoRef.current.muted = false;
        setIsMuted(false);
      } else if (val === 0 && !isMuted) {
        videoRef.current.muted = true;
        setIsMuted(true);
      }
    }
  };

  const setQuality = (level: number) => {
    if (hlsRef.current) {
      hlsRef.current.currentLevel = level;
      setShowQualityMenu(false);
    }
  };

  const toggleFullscreen = () => {
    const container = videoRef.current?.parentElement;
    if (!container) return;
    if (!document.fullscreenElement) {
      container.requestFullscreen();
    } else {
      document.exitFullscreen();
    }
  };

  // categories definition removed to avoid duplication

  return (
    <div className="flex-1 p-4 md:p-6 overflow-y-auto">
      {/* VIDEO PLAYER */}
      <div className={`aspect-video bg-black mb-6 flex items-center justify-center border shadow-2xl relative overflow-hidden group ${
        liquidGlass ? "rounded-2xl" : "rounded-lg"
      } ${isDark ? "border-slate-800" : "border-slate-300"}`}>
        {!user && !isDev ? (
          <div className={`absolute inset-0 z-10 flex flex-col items-center justify-center bg-slate-900/40 p-6 text-center ${
            liquidGlass ? "backdrop-blur-xl" : "backdrop-blur-none"
          }`}>
            <motion.div 
              initial={{ scale: 0.8, opacity: 0 }}
              animate={{ scale: 1, opacity: 1 }}
              className={`p-10 border shadow-2xl flex flex-col items-center space-y-6 bg-white/80 border-black/5 ${
                liquidGlass ? "rounded-[40px]" : "rounded-2xl"
              }`}
            >
              <div className="p-4 rounded-full bg-primary/10">
                <Lock className="h-10 w-10 text-primary" />
              </div>
              <div className="space-y-1">
                <h3 className="text-2xl font-bold text-slate-900">Đăng nhập để xem</h3>
                <p className="text-slate-500 text-sm max-w-[280px]">Vui lòng đăng nhập tài khoản VPlay để có thể xem kênh trực tuyến này.</p>
              </div>
              <button 
                onClick={onLogin}
                className="w-full py-4 bg-primary hover:opacity-90 text-white rounded-3xl font-bold transition-all hover:scale-105 active:scale-95 shadow-lg shadow-primary/20"
              >
                Đăng nhập ngay
              </button>
            </motion.div>
          </div>
        ) : active.status === "maintenance" ? (
          <div className={`absolute inset-0 flex flex-col items-center justify-center ${liquidGlass ? "bg-black/20 backdrop-blur-xl" : "bg-black"}`}>
            <img 
              src="https://upload.wikimedia.org/wikipedia/commons/5/5b/EBU_Colorbars_HD.svg" 
              alt="Maintenance" 
              className={`w-full h-full object-cover ${liquidGlass ? "opacity-40" : "opacity-80"}`}
              referrerPolicy="no-referrer"
            />
            <div className={`absolute inset-0 flex flex-col items-center justify-center ${liquidGlass ? "bg-white/5" : "bg-black/40"} backdrop-blur-[2px]`}>
              <div className={`${liquidGlass ? "bg-amber-500/80 backdrop-blur-md" : "bg-amber-500"} text-white px-6 py-3 rounded-2xl font-black text-xl shadow-2xl animate-pulse border-2 border-white/20 uppercase`}>
                {active.statusLabel || "Trực tiếp"}
              </div>
              <p className="text-white/80 mt-4 font-medium text-sm drop-shadow-md">Vui lòng quay lại sau</p>
            </div>
          </div>
        ) : (
          <>
            <video
              ref={videoRef}
              className="w-full h-full"
              autoPlay
              muted={isMuted}
              onClick={togglePlay}
            />
            {/* Buffering Indicator */}
            <AnimatePresence>
              {isBuffering && isPlaying && (
                <motion.div
                  initial={{ opacity: 0 }}
                  animate={{ opacity: 1 }}
                  exit={{ opacity: 0 }}
                  className="absolute inset-0 flex flex-col items-center justify-center bg-black/20 backdrop-blur-[2px] z-20 pointer-events-none"
                >
                  <div className="relative">
                    <motion.div
                      animate={{ rotate: 360 }}
                      transition={{ duration: 1, repeat: Infinity, ease: "linear" }}
                      className="w-12 h-12 border-4 border-white/20 border-t-primary rounded-full"
                    />
                    <motion.div
                      animate={{ scale: [1, 1.2, 1] }}
                      transition={{ duration: 2, repeat: Infinity }}
                      className="absolute inset-0 flex items-center justify-center"
                    >
                      <div className="w-2 h-2 bg-primary rounded-full" />
                    </motion.div>
                  </div>
                  <p className="text-white text-[10px] font-black uppercase tracking-[0.2em] mt-4 drop-shadow-lg opacity-80">
                    Đang tải luồng...
                  </p>
                </motion.div>
              )}
            </AnimatePresence>
            {/* Tap to Unmute Overlay */}
            {isMuted && isPlaying && (
              <button 
                onClick={toggleMute}
                className="absolute top-4 right-4 bg-black/60 backdrop-blur-md text-white px-3 py-1.5 rounded-full text-xs font-bold flex items-center gap-2 hover:bg-black/80 transition-all animate-bounce"
              >
                <VolumeX className="h-4 w-4" />
                CHẠM ĐỂ BẬT TIẾNG
              </button>
            )}
            {/* Modern Control Bar */}
            <div className="absolute inset-x-0 bottom-0 p-4 bg-gradient-to-t from-black/80 to-transparent opacity-0 group-hover:opacity-100 transition-opacity duration-300 flex items-center justify-between">
              <div className="flex items-center gap-4">
                <button onClick={togglePlay} className="text-white hover:text-primary transition-colors">
                  {isPlaying ? <Pause className="h-8 w-8 fill-white" /> : <Play className="h-8 w-8 fill-white" />}
                </button>
                <div className="flex items-center gap-2 group/volume">
                  <button onClick={toggleMute} className="text-white hover:text-primary transition-colors">
                    {isMuted || volume === 0 ? <VolumeX className="h-6 w-6" /> : <Volume2 className="h-6 w-6" />}
                  </button>
                  <input 
                    type="range" 
                    min="0" 
                    max="1" 
                    step="0.05" 
                    value={isMuted ? 0 : volume} 
                    onChange={handleVolumeChange}
                    className="w-0 group-hover/volume:w-20 transition-all duration-300 h-1 bg-slate-600 rounded-lg appearance-none cursor-pointer accent-primary"
                  />
                </div>
              </div>
              <div className="flex gap-4">
                <button onClick={toggleFullscreen} className="text-white hover:text-primary transition-colors">
                  <Maximize className="h-6 w-6" />
                </button>
                <div className="relative">
                  <button onClick={() => setShowQualityMenu(!showQualityMenu)} className="text-white hover:text-primary transition-colors">
                    <SettingsIcon className="h-6 w-6" />
                  </button>
                  {showQualityMenu && (
                    <motion.div 
                      initial={{ opacity: 0, scale: 0.9, y: 10 }}
                      animate={{ opacity: 1, scale: 1, y: 0 }}
                      className="absolute bottom-12 right-0 bg-white/90 backdrop-blur-xl rounded-2xl p-2 text-sm text-slate-900 border border-white/60 w-32 shadow-2xl"
                    >
                      <button onClick={() => setQuality(-1)} className={`block w-full text-left px-4 py-2 hover:bg-black/5 rounded-xl transition-colors ${currentLevel === -1 ? "text-primary font-bold" : ""}`}>Tự động</button>
                      {levels.map((level, index) => (
                        <button key={index} onClick={() => setQuality(index)} className={`block w-full text-left px-4 py-2 hover:bg-black/5 rounded-xl transition-colors ${currentLevel === index ? "text-primary font-bold" : ""}`}>
                          {level.height}p
                        </button>
                      ))}
                    </motion.div>
                  )}
                </div>
              </div>
            </div>
          </>
        )}
      </div>

      {/* CHANNEL INFO */}
      <div className="flex flex-col md:flex-row md:items-center justify-between gap-4">
        <div className="flex items-center gap-4">
          <h2 className={`text-2xl font-bold flex items-center gap-3 ${isDark ? "text-white" : "text-slate-950"}`}>
            {active.name}
            {active.status === "maintenance" ? (
              <span className="bg-amber-500 text-white text-xs px-2 py-0.5 rounded-full font-bold flex items-center gap-1">
                <span className="h-1.5 w-1.5 rounded-full bg-white"></span>
                {active.statusLabel || "TRỰC TIẾP"}
              </span>
            ) : (
              <span className="bg-red-600 text-white text-xs px-2 py-0.5 rounded-full font-bold flex items-center gap-1">
                <span className="h-1.5 w-1.5 rounded-full bg-white animate-pulse"></span>
                BÌNH THƯỜNG
              </span>
            )}
          </h2>
          <button 
            onClick={() => toggleFavorite(active)}
            className={`p-2 rounded-full transition-all hover:scale-110 ${favorites.includes(active.name) ? "text-red-500" : isDark ? "text-slate-500 hover:text-slate-300" : "text-slate-400 hover:text-slate-600"}`}
          >
            <Heart className={`h-6 w-6 ${favorites.includes(active.name) ? "fill-red-500" : ""}`} />
          </button>
        </div>
      </div>

      {/* FILTERS */}
      <div className="mt-8">
        <div className="flex flex-col md:flex-row gap-4 mb-8">
          <div className="flex gap-2 overflow-x-auto pb-2 md:pb-0 scrollbar-hide flex-1">
            {["Tất cả", "VTV", "HTV", "VTVcab", "Thiết yếu", "Phim & Video", "Địa phương", "Phát thanh", "Hoạt động", "Trực tiếp"].map((type) => (
              <button
                key={type}
                onClick={() => setFilterType(type)}
                className={`px-5 py-2.5 md:px-4 md:py-2 rounded-full text-sm font-medium whitespace-nowrap transition-all ${
                  filterType === type
                    ? "bg-primary text-white shadow-lg shadow-primary/20"
                    : isDark
                    ? "bg-white/5 border-white/5 text-slate-400 hover:bg-white/10"
                    : "bg-white/10 border-white/20 text-slate-600 hover:bg-white/20"
                } ${liquidGlass ? "backdrop-blur-md" : ""}`}
              >
                {type}
              </button>
            ))}
          </div>

          <div className="flex gap-2">
            {/* Desktop Sort Dropdown */}
            <div className="relative hidden md:block">
              <button
                onClick={() => setShowSortMenu(!showSortMenu)}
                className={`p-3 md:p-3 rounded-xl border transition-all flex items-center gap-2 ${
                  isDark 
                    ? "bg-slate-800/50 border-slate-700/50 text-white" 
                    : "bg-white/50 border-white/60 text-slate-900"
                } ${liquidGlass ? "backdrop-blur-md" : ""}`}
              >
                <Sliders className="h-5 w-5" />
                <span className="text-sm font-medium">
                  {sortOrder === "default" ? "Mặc định" : sortOrder === "az" ? "A-Z" : "Z-A"}
                </span>
                <ChevronLeft className={`h-4 w-4 transition-transform ${showSortMenu ? "-rotate-90" : ""}`} />
              </button>
              
              <AnimatePresence>
                {showSortMenu && (
                  <motion.div
                    initial={{ opacity: 0, scale: 0.95, y: 10 }}
                    animate={{ opacity: 1, scale: 1, y: 0 }}
                    exit={{ opacity: 0, scale: 0.95, y: 10 }}
                    className={`absolute top-full right-0 mt-2 z-50 p-2 border shadow-2xl w-48 ${
                      isDark ? "bg-slate-900/95 border-white/10" : "bg-white/95 border-black/5"
                    } ${liquidGlass ? "rounded-2xl backdrop-blur-3xl" : "rounded-xl"}`}
                  >
                    {[
                      { id: "default", label: "Mặc định" },
                      { id: "az", label: "Sắp xếp A-Z" },
                      { id: "za", label: "Sắp xếp Z-A" }
                    ].map((opt) => (
                      <button
                        key={opt.id}
                        onClick={() => {
                          setSortOrder(opt.id as any);
                          setShowSortMenu(false);
                        }}
                        className={`w-full text-left px-4 py-3 rounded-xl text-sm font-bold transition-all ${
                          sortOrder === opt.id 
                            ? "bg-primary text-white" 
                            : isDark ? "text-white hover:bg-white/5" : "text-slate-900 hover:bg-black/5"
                        }`}
                      >
                        {opt.label}
                      </button>
                    ))}
                  </motion.div>
                )}
              </AnimatePresence>
            </div>

            {/* Mobile Sort Dropdown */}
            <div className="relative md:hidden flex-1">
              <button
                onClick={() => setShowSortMenu(!showSortMenu)}
                className={`w-full p-3.5 rounded-xl border transition-all flex items-center justify-center gap-2 ${
                  isDark 
                    ? "bg-white/5 border-white/5 text-white" 
                    : "bg-white/10 border-white/20 text-slate-900"
                } ${liquidGlass ? "backdrop-blur-md" : ""}`}
              >
                <Sliders className="h-5 w-5" />
                <span className="text-sm font-bold uppercase tracking-wider">Sort</span>
                <span className="ml-auto text-[10px] opacity-50">
                  {sortOrder === "default" ? "Mặc định" : sortOrder === "az" ? "A-Z" : "Z-A"}
                </span>
              </button>
              
              <AnimatePresence>
                {showSortMenu && (
                  <motion.div
                    initial={{ opacity: 0, y: 10, scale: 0.95 }}
                    animate={{ opacity: 1, y: 0, scale: 1 }}
                    exit={{ opacity: 0, y: 10, scale: 0.95 }}
                    className={`absolute top-full left-0 right-0 mt-2 z-50 p-2 border shadow-2xl ${
                      isDark ? "bg-slate-900/95 border-white/10" : "bg-white/95 border-black/5"
                    } ${liquidGlass ? "rounded-2xl backdrop-blur-3xl" : "rounded-xl"}`}
                  >
                    {[
                      { id: "default", label: "Mặc định" },
                      { id: "az", label: "Sắp xếp A-Z" },
                      { id: "za", label: "Sắp xếp Z-A" }
                    ].map((opt) => (
                      <button
                        key={opt.id}
                        onClick={() => {
                          setSortOrder(opt.id as any);
                          setShowSortMenu(false);
                        }}
                        className={`w-full text-left px-4 py-3 rounded-xl text-sm font-bold transition-all ${
                          sortOrder === opt.id 
                            ? "bg-primary text-white" 
                            : isDark ? "text-white hover:bg-white/5" : "text-slate-900 hover:bg-black/5"
                        }`}
                      >
                        {opt.label}
                      </button>
                    ))}
                  </motion.div>
                )}
              </AnimatePresence>
            </div>
          </div>
        </div>

        {/* CHANNEL LIST */}
        <div className="space-y-8">
          {filteredCategories.map(cat => (
            <div key={cat}>
              <h3 className={`mb-4 text-lg font-semibold ${isDark ? "text-slate-200" : "text-slate-800"}`}>{cat}</h3>
              <div className="grid grid-cols-3 sm:grid-cols-3 md:grid-cols-6 gap-3 md:gap-4">
                {cat === "Phát thanh" ? (
                  <div className={`col-span-full p-8 rounded-2xl border-2 border-dashed flex flex-col items-center justify-center gap-3 ${
                    isDark ? "border-white/10 bg-white/5 text-slate-400" : "border-black/5 bg-black/5 text-slate-500"
                  }`}>
                    <Sparkles className="w-8 h-8 animate-pulse text-primary" />
                    <p className="font-bold text-lg tracking-widest uppercase">Coming Soon!</p>
                    <p className="text-xs opacity-60">Tính năng đang được phát triển</p>
                  </div>
                ) : (
                  filteredChannels.filter(c => c.category === cat).map((ch) => (
                    <ChannelCard 
                      key={`${ch.name}-${ch.stream}`} 
                      ch={ch} 
                      onClick={() => setActive(ch)} 
                      isDark={isDark} 
                      isActive={active.name === ch.name} 
                      favorites={favorites} 
                      toggleFavorite={toggleFavorite} 
                      liquidGlass={liquidGlass}
                    />
                  ))
                )}
              </div>
            </div>
          ))}
          {filteredChannels.length === 0 && (
            <div className="text-center py-20">
              <div className="inline-flex items-center justify-center w-20 h-20 rounded-full bg-slate-800/50 mb-4">
                <img 
                  src="https://static.wikia.nocookie.net/ftv/images/6/63/Search_uci.png/revision/latest?cb=20260411084053&path-prefix=vi" 
                  alt="Search" 
                  className="h-10 w-10 object-contain" 
                  referrerPolicy="no-referrer" 
                />
              </div>
              <h3 className="text-xl font-bold text-slate-400">Không tìm thấy kênh nào</h3>
              <p className="text-slate-500">Thử tìm kiếm với từ khóa khác</p>
            </div>
          )}
        </div>
      </div>
    </div>
  );
}

function SearchPopup({ 
  isDark, 
  searchQuery, 
  setActiveChannel, 
  onClose, 
  favorites, 
  liquidGlass,
  setActiveTab,
  setIsDark,
  setLiquidGlass,
  onLogin,
  onLogout,
  setSortOrder
}: {
  isDark: boolean,
  searchQuery: string,
  setActiveChannel: (ch: typeof channels[0]) => void,
  onClose: () => void,
  favorites: string[],
  liquidGlass: boolean,
  setActiveTab: (tab: string) => void,
  setIsDark: (val: boolean) => void,
  setLiquidGlass: (val: boolean) => void,
  onLogin: () => void,
  onLogout: () => void,
  setSortOrder: (val: "az" | "za") => void
}) {
  if (searchQuery.trim() === "") return null;

  const filteredChannels = channels.filter(ch => 
    ch.name.toLowerCase().includes(searchQuery.toLowerCase()) ||
    ch.category.toLowerCase().includes(searchQuery.toLowerCase())
  );

  const systemItems = [
    { name: "Trang chủ", type: "tab", icon: Home, action: () => setActiveTab("Trang chủ") },
    { name: "Phát sóng", type: "tab", icon: Tv, action: () => setActiveTab("Phát sóng") },
    { name: "Lịch phát sóng", type: "tab", icon: Calendar, action: () => setActiveTab("Lịch phát sóng") },
    { name: "Phát lại", type: "tab", icon: History, action: () => setActiveTab("Phát lại") },
    { name: "Phát thanh", type: "tab", icon: Radio, action: () => setActiveTab("Phát thanh") },
    { name: "Cài đặt", type: "tab", icon: SettingsIcon, action: () => setActiveTab("Cài đặt") },
    { name: "Hồ sơ", type: "tab", icon: User, action: () => setActiveTab("Hồ sơ") },
    { name: "Chế độ tối", type: "setting", icon: Moon, action: () => setIsDark(!isDark) },
    { name: "Hiệu ứng kính", type: "setting", icon: Layers, action: () => setLiquidGlass(!liquidGlass) },
    { name: "Đăng nhập", type: "button", icon: LogIn, action: onLogin },
    { name: "Đăng xuất", type: "button", icon: LogOut, action: onLogout },
    { name: "Sắp xếp A-Z", type: "toggle", icon: Filter, action: () => setSortOrder("az") },
    { name: "Sắp xếp Z-A", type: "toggle", icon: Filter, action: () => setSortOrder("za") },
  ];

  const filteredSystem = systemItems.filter(item => 
    item.name.toLowerCase().includes(searchQuery.toLowerCase())
  );

  const favoriteChannels = channels.filter(ch => favorites.includes(ch.name));

  return (
    <motion.div
      initial={{ opacity: 0, y: 40, scale: 0.8, rotateX: -15 }}
      animate={{ opacity: 1, y: 0, scale: 1, rotateX: 0 }}
      exit={{ opacity: 0, y: 40, scale: 0.8, rotateX: -15 }}
      transition={{ type: "spring", damping: 20, stiffness: 300 }}
      className={`absolute bottom-full mb-6 w-[90vw] md:w-full max-w-[400px] border shadow-[0_20px_50px_rgba(0,0,0,0.3)] overflow-hidden ${
        liquidGlass ? "rounded-[32px] backdrop-blur-3xl" : "rounded-xl backdrop-blur-none"
      } bg-white/95 border-white/80 shadow-2xl`}
    >
      <div className="p-4 space-y-1 max-h-[60vh] overflow-y-auto">
        {searchQuery.trim() === "" ? (
          <div className="space-y-4">
            {favoriteChannels.length > 0 && (
              <div className="space-y-2">
                <div className="px-4 py-2 flex items-center gap-2">
                  <Heart className="w-3 h-3 text-red-500 fill-red-500" />
                  <p className={`text-[10px] font-bold uppercase tracking-widest text-black/60`}>Kênh yêu thích</p>
                </div>
                {favoriteChannels.map(ch => (
                  <button
                    key={ch.name}
                    onClick={() => { setActiveChannel(ch); onClose(); }}
                    className={`w-full flex items-center gap-4 p-3 rounded-[24px] transition-all hover:scale-[1.02] active:scale-[0.98] group hover:bg-black/5`}
                  >
                    <div className={`w-10 h-10 rounded-xl flex items-center justify-center border bg-slate-100 border-slate-200`}>
                      <img src={ch.logo} alt={ch.name} className="w-6 h-6 object-contain" referrerPolicy="no-referrer" />
                    </div>
                    <div className="flex-1 text-left">
                      <p className={`font-bold text-sm text-black`}>{ch.name}</p>
                    </div>
                    <ChevronRight className="w-4 h-4 text-black/30" />
                  </button>
                ))}
              </div>
            )}
            <div className="py-8 text-center space-y-3 text-black">
              <img 
                src="https://static.wikia.nocookie.net/ftv/images/6/63/Search_uci.png/revision/latest?cb=20260411084053&path-prefix=vi" 
                alt="Search" 
                className="w-12 h-12 mx-auto object-contain" 
                referrerPolicy="no-referrer" 
              />
              <p className="text-sm font-bold">Tìm kiếm kênh chương trình</p>
            </div>
          </div>
        ) : (filteredChannels.length > 0 || filteredSystem.length > 0) ? (
          <>
            {filteredSystem.length > 0 && (
              <div className="space-y-1 mb-4">
                <div className="px-4 py-2">
                  <p className={`text-[10px] font-bold uppercase tracking-widest text-black/60`}>Hệ thống & Cài đặt</p>
                </div>
                {filteredSystem.map(item => (
                  <button
                    key={item.name}
                    onClick={() => { item.action(); onClose(); }}
                    className={`w-full flex items-center gap-4 p-3 rounded-[24px] transition-all hover:scale-[1.02] active:scale-[0.98] group hover:bg-black/5`}
                  >
                    <div className={`w-12 h-12 rounded-2xl flex items-center justify-center border transition-transform group-hover:rotate-3 bg-slate-100 border-slate-200 text-primary`}>
                      <item.icon className="w-6 h-6 fill-current" />
                    </div>
                    <div className="flex-1 text-left">
                      <p className={`font-bold text-sm text-black`}>{item.name}</p>
                      <p className="text-[10px] font-bold uppercase tracking-wider text-black/60">{item.type === "tab" ? "Chuyển Tab" : "Cài đặt"}</p>
                    </div>
                    <ChevronRight className="w-4 h-4 text-black/30" />
                  </button>
                ))}
              </div>
            )}

            {filteredChannels.length > 0 && (
              <div className="space-y-1">
                <div className="px-4 py-2">
                  <p className={`text-[10px] font-bold uppercase tracking-widest text-black/60`}>Kênh truyền hình</p>
                </div>
                {filteredChannels.map(ch => (
                  <button
                    key={ch.name}
                    onClick={() => { setActiveChannel(ch); onClose(); }}
                    className={`w-full flex items-center gap-4 p-3 rounded-[24px] transition-all hover:scale-[1.02] active:scale-[0.98] group hover:bg-black/5`}
                  >
                    <div className={`w-12 h-12 rounded-2xl flex items-center justify-center border transition-transform group-hover:rotate-3 bg-slate-100 border-slate-200`}>
                      <img src={ch.logo} alt={ch.name} className="w-8 h-8 object-contain" referrerPolicy="no-referrer" />
                    </div>
                    <div className="flex-1 text-left">
                      <p className={`font-bold text-sm text-black`}>{ch.name}</p>
                      <p className="text-[10px] font-bold uppercase tracking-wider text-black/60">{ch.category}</p>
                    </div>
                    <ChevronRight className="w-4 h-4 text-black/30" />
                  </button>
                ))}
              </div>
            )}
          </>
        ) : (
          <div className="py-12 text-center opacity-40 space-y-3 text-black">
            <img 
              src="https://static.wikia.nocookie.net/ftv/images/6/63/Search_uci.png/revision/latest?cb=20260411084053&path-prefix=vi" 
              alt="Search" 
              className="w-12 h-12 mx-auto object-contain" 
              referrerPolicy="no-referrer" 
            />
            <p className="text-sm font-medium">Không tìm thấy kết quả nào cho "{searchQuery}"</p>
          </div>
        )}
      </div>
    </motion.div>
  );
}

function EventsContent({ isDark, liquidGlass }: { isDark: boolean, liquidGlass: boolean }) {
  const events = [
    { title: "Giải Ngoại Hạng Anh", date: "20:00 - 15/04", channel: "K+ Sport 1", type: "Thể thao" },
    { title: "Chung Kết Rap Việt", date: "21:00 - 16/04", channel: "HTV2 - Vie Channel", type: "Giải trí" },
    { title: "Lễ Trao Giải Oscar", date: "07:00 - 18/04", channel: "VTV3", type: "Sự kiện" },
    { title: "Đại Nhạc Hội Vplay", date: "19:00 - 20/04", channel: "Vplay Live", type: "Đặc sắc" },
    { title: "Chung kết Giọng hát Vplay", date: "20:00 - 25/04", channel: "Vplay Live", type: "Đặc sắc" },
    { title: "GALA Vplay 2027", date: "19:00 - 31/12", channel: "Vplay Live", type: "Đặc sắc" },
    { title: "Giải bóng đá vô địch thế giới Vplay Cup 2026", date: "21:00 - 15/06", channel: "Vplay Sports", type: "Thể thao" },
    { title: "V-play Concert Đặc Biệt", date: "20:30 - 01/05", channel: "Vplay Live", type: "Đặc sắc" },
  ];

  return (
    <div className="p-4 md:p-8 max-w-4xl mx-auto space-y-8">
      <div className="flex items-center gap-3 mb-6">
        <Sparkles className="w-8 h-8 text-primary" />
        <h2 className={`text-3xl font-bold ${isDark ? "text-white" : "text-slate-900"}`}>Sự kiện sắp tới</h2>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
        {events.map((event, i) => (
          <motion.div
            key={i}
            initial={{ opacity: 0, y: 20 }}
            animate={{ opacity: 1, y: 0 }}
            transition={{ delay: i * 0.1 }}
            className={`p-6 border ${
              liquidGlass ? "rounded-3xl backdrop-blur-xl" : "rounded-2xl"
            } ${
              isDark 
                ? "bg-white/5 border-white/10 hover:bg-white/10" 
                : "bg-white border-slate-100 shadow-sm hover:shadow-md"
            } transition-all group cursor-pointer`}
          >
            <div className="flex justify-between items-start mb-4">
              <span className={`px-3 py-1 rounded-full text-[10px] font-bold uppercase tracking-wider ${
                isDark ? "bg-primary/20 text-primary" : "bg-primary/10 text-primary"
              }`}>
                {event.type}
              </span>
              <Calendar className="w-5 h-5 opacity-30" />
            </div>
            <h3 className={`text-xl font-bold mb-2 ${isDark ? "text-white" : "text-slate-900"}`}>{event.title}</h3>
            <div className="space-y-1 opacity-60 text-sm">
              <p className="flex items-center gap-2">
                <Clock className="w-4 h-4" />
                {event.date}
              </p>
              <p className="flex items-center gap-2">
                <Tv className="w-4 h-4" />
                {event.channel}
              </p>
            </div>
            <div className="mt-6 flex justify-end">
              <button className={`px-4 py-2 rounded-xl text-xs font-bold transition-all ${
                isDark ? "bg-primary text-white" : "bg-primary text-white"
              } group-hover:scale-105 active:scale-95`}>
                Đặt nhắc hẹn
              </button>
            </div>
          </motion.div>
        ))}
      </div>
    </div>
  );
}

function ScheduleContent({ isDark, liquidGlass }: { isDark: boolean, liquidGlass: boolean }) {
  const hours = Array.from({ length: 24 }, (_, i) => i);
  const hourWidth = 250;

  const MOCK_PROGRAMS: Record<string, any[]> = {
    "VTV1": [
      { id: 1, title: "Chào buổi sáng", start: 6, duration: 120, color: "bg-blue-500" },
      { id: 2, title: "Thời sự trưa", start: 12, duration: 60, color: "bg-sky-500" },
      { id: 3, title: "Phim truyện chiều", start: 14.5, duration: 90, color: "bg-rose-500" },
      { id: 4, title: "Thời sự 19h", start: 19, duration: 45, color: "bg-indigo-600" },
      { id: 5, title: "Gala Cười 2026", start: 20, duration: 120, color: "bg-amber-600" },
    ],
    "VTV3": [
      { id: 6, title: "Café sáng", start: 7, duration: 60, color: "bg-orange-500" },
      { id: 7, title: "Đường lên đỉnh Olympia", start: 13, duration: 90, color: "bg-emerald-600" },
      { id: 8, title: "Vietnam Idol", start: 20, duration: 150, color: "bg-purple-600" },
    ],
    "HTV7": [
      { id: 9, title: "Running Man Việt Nam", start: 20.5, duration: 120, color: "bg-red-500" },
      { id: 10, title: "Hành trình ẩm thực", start: 15, duration: 60, color: "bg-yellow-500" },
    ],
    "HTV9": [
      { id: 11, title: "Siêu trí tuệ", start: 21, duration: 90, color: "bg-cyan-600" },
    ],
    "Vĩnh Long 1 (THVL1)": [
      { id: 12, title: "Cánh đồng hoang", start: 19.5, duration: 120, color: "bg-teal-600" },
    ]
  };

  const scheduleChannelsList = channels.filter(ch => ["VTV1", "VTV2", "VTV3", "HTV7", "HTV9", "Vĩnh Long 1 (THVL1)", "An Giang 1 (ATV1)"].includes(ch.name));

  return (
    <div className="flex-1 flex flex-col min-h-0">
      <div className="p-4 md:p-8 flex items-center justify-between gap-4">
        <div className="flex items-center gap-3">
          <Calendar className="w-8 h-8 text-primary" />
          <h2 className={`text-3xl font-bold ${isDark ? "text-white" : "text-slate-900"}`}>Lịch phát sóng</h2>
        </div>
        <div className={`p-2 rounded-xl text-xs font-bold uppercase tracking-wider ${isDark ? "bg-white/5 text-slate-400" : "bg-black/5 text-slate-500"}`}>
          Hôm nay, {new Date().toLocaleDateString('vi-VN')}
        </div>
      </div>

      <div className="flex-1 min-h-0 bg-inherit md:p-8 md:pt-0 overflow-hidden">
        <div className={`h-full border flex flex-col ${
          isDark ? "bg-slate-900/50 border-white/5" : "bg-white border-black/5"
        } ${liquidGlass ? "rounded-[40px] backdrop-blur-xl" : "rounded-3xl"} overflow-hidden shadow-2xl relative`}>
          
          <div className="flex-1 overflow-auto scrollbar-thin scrollbar-thumb-primary/20">
            <div className="min-w-max relative flex flex-col">
              
              {/* Timeline Header */}
              <div className="flex sticky top-0 z-40">
                <div className={`sticky left-0 z-50 w-32 md:w-56 h-16 border-r border-b flex items-center px-4 md:px-8 font-black text-[10px] md:text-xs uppercase tracking-[0.2em] opacity-30 ${
                  isDark ? "bg-[#0b1221] border-white/10" : "bg-slate-50 border-black/5"
                }`}>
                  Kênh
                </div>
                {hours.map(h => (
                  <div key={h} className={`w-[250px] h-16 border-b border-r flex items-center justify-center font-black text-sm md:text-base opacity-40 transition-opacity hover:opacity-100 ${
                    isDark ? "bg-[#0b1221] border-white/10" : "bg-slate-50 border-black/5"
                  }`}>
                    {h.toString().padStart(2, '0')}:00
                  </div>
                ))}
              </div>

              {/* Rows */}
              <div className="flex-1 flex flex-col">
                {scheduleChannelsList.map((ch) => (
                  <div key={ch.name} className="flex relative h-28 md:h-32 group">
                    {/* Sticky Channel Column */}
                    <div className={`sticky left-0 z-20 w-32 md:w-56 h-full border-r border-b flex items-center px-4 md:px-8 gap-3 md:gap-4 group-hover:bg-primary/5 transition-all duration-300 ${
                      isDark ? "bg-[#0b1221] border-white/10" : "bg-white border-black/5"
                    }`}>
                      <div className="w-10 h-10 md:w-12 md:h-12 flex-shrink-0 bg-white/5 rounded-xl flex items-center justify-center p-2">
                        <img src={ch.logo} alt="" className="w-full h-full object-contain" referrerPolicy="no-referrer" />
                      </div>
                      <span className={`font-black text-[10px] md:text-xs uppercase tracking-wider truncate transition-colors group-hover:text-primary ${
                        isDark ? "text-white" : "text-slate-900"
                      }`}>
                        {ch.name}
                      </span>
                    </div>

                    {/* Timeline Data */}
                    <div className="flex relative items-center">
                      {/* Grid Lines */}
                      {hours.map(h => (
                        <div key={h} className={`w-[250px] h-full border-r border-b opacity-5 pointer-events-none ${
                          isDark ? "border-white" : "border-black"
                        }`} />
                      ))}

                      {/* Programs */}
                      <div className="absolute inset-y-0 left-0 flex items-center px-2">
                        {(MOCK_PROGRAMS[ch.name] || []).map(prog => (
                          <motion.div
                            key={prog.id}
                            whileHover={{ scale: 1.02, y: -2 }}
                            whileTap={{ scale: 0.98 }}
                            className={`absolute h-20 md:h-24 rounded-2xl md:rounded-3xl p-3 md:p-5 flex flex-col justify-center border border-white/20 shadow-xl overflow-hidden cursor-pointer group/prog ${prog.color}`}
                            style={{
                              left: (prog.start * hourWidth) + 8,
                              width: (prog.duration / 60) * hourWidth - 16
                            }}
                          >
                            <div className="absolute inset-0 bg-gradient-to-br from-white/20 to-transparent pointer-events-none" />
                            <h4 className="text-white font-black text-xs md:text-lg line-clamp-1 drop-shadow-md leading-tight">
                              {prog.title}
                            </h4>
                            <div className="flex items-center gap-2 mt-1 opacity-80">
                              <Clock size={10} className="md:w-3 md:h-3" />
                              <span className="text-[8px] md:text-[11px] font-bold uppercase tracking-wider">
                                {Math.floor(prog.start).toString().padStart(2, '0')}:{(prog.start % 1 * 60).toString().padStart(2, '0')} · {prog.duration} phút
                              </span>
                            </div>
                          </motion.div>
                        ))}
                      </div>
                    </div>
                  </div>
                ))}
              </div>
            </div>
          </div>
          
          {/* Scroll Hint Overlay */}
          <div className={`absolute bottom-6 right-6 p-4 rounded-3xl border border-white/10 flex items-center gap-3 backdrop-blur-3xl shadow-2xl pointer-events-none md:hidden ${
            isDark ? "bg-slate-950/80 text-white" : "bg-white/80 text-slate-900"
          }`}>
             <MousePointer2 className="w-5 h-5 animate-bounce" />
             <span className="text-[10px] font-black uppercase tracking-widest">Trượt ngang xem tiếp</span>
          </div>
        </div>
      </div>
    </div>
  );
}

function PlaybackContent({ 
  isDark, 
  liquidGlass, 
  user, 
  setActiveChannel, 
  setActiveTab,
  favorites,
  toggleFavorite
}: { 
  isDark: boolean, 
  liquidGlass: boolean, 
  user: any,
  setActiveChannel: (ch: Channel) => void,
  setActiveTab: (tab: string) => void,
  favorites: string[],
  toggleFavorite: (ch: Channel) => void
}) {
  const [history, setHistory] = useState<Channel[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (!user) {
      setLoading(false);
      return;
    }

    const userRef = doc(db, "users", user.uid);
    const unsub = onSnapshot(userRef, (snapshot) => {
      if (snapshot.exists()) {
        const data = snapshot.data();
        const watchedNames = data.watchedChannels || [];
        // Map names back to channel objects
        const watchedChannels = (watchedNames as string[]).map((name: string) => 
          channels.find(c => c.name === name)
        ).filter(Boolean) as Channel[];
        
        // Watched names are deduplicated by arrayUnion, but let's reverse to show "recently added" at top
        setHistory([...watchedChannels].reverse());
      }
      setLoading(false);
    }, (err) => {
      console.error("Playback fetch error:", err);
      setLoading(false);
    });

    return () => unsub();
  }, [user]);

  if (!user) {
    return (
      <div className="p-8 text-center flex flex-col items-center justify-center min-h-[50vh] space-y-4">
        <div className="w-16 h-16 bg-slate-100 rounded-full flex items-center justify-center text-slate-400">
          <History size={32} />
        </div>
        <h2 className={`text-xl font-bold ${isDark ? "text-white" : "text-slate-900"}`}>Đăng nhập để xem lịch sử</h2>
        <p className="text-sm opacity-60">Bạn cần đăng nhập để lưu trữ và xem lại các kênh đã xem.</p>
      </div>
    );
  }

  if (loading) return <div className="p-8 text-center">Đang tải lịch sử...</div>;

  return (
    <div className="p-4 md:p-8 max-w-6xl mx-auto space-y-8">
      <div className="flex items-center gap-3 mb-6">
        <History className="w-8 h-8 text-primary" />
        <h2 className={`text-3xl font-bold ${isDark ? "text-white" : "text-slate-900"}`}>Phát lại gần đây</h2>
      </div>

      {history.length > 0 ? (
        <div className="grid grid-cols-3 sm:grid-cols-3 md:grid-cols-6 gap-3 md:gap-4">
          {history.map((ch, i) => (
            <ChannelCard 
              key={`${ch.name}-${i}`} 
              ch={ch} 
              onClick={() => {
                setActiveChannel(ch);
                setActiveTab("Phát sóng");
              }}
              isDark={isDark}
              favorites={favorites}
              toggleFavorite={toggleFavorite}
              liquidGlass={liquidGlass}
            />
          ))}
        </div>
      ) : (
        <div className="text-center py-20 opacity-40 italic">
          Bạn chưa xem kênh nào gần đây.
        </div>
      )}
    </div>
  );
}

function VideoContent({ isDark, liquidGlass, setActiveChannel, setActiveTab }: { 
  isDark: boolean, 
  liquidGlass: boolean,
  setActiveChannel: (ch: Channel) => void,
  setActiveTab: (tab: string) => void
}) {
  const videoChannels = channels.filter(ch => ch.category === "Phim & Video" || ch.status === "movie");

  return (
    <div className="p-4 md:p-8 max-w-6xl mx-auto space-y-8">
      <div className="flex items-center gap-3 mb-6">
        <PlayCircle className="w-8 h-8 text-primary" />
        <h2 className={`text-3xl font-bold ${isDark ? "text-white" : "text-slate-900"}`}>Video & Phim</h2>
      </div>

      <div className="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-6">
        {videoChannels.map((video, i) => (
          <motion.div
            key={i}
            initial={{ opacity: 0, scale: 0.9 }}
            animate={{ opacity: 1, scale: 1 }}
            transition={{ delay: i * 0.1 }}
            onClick={() => {
              setActiveChannel(video);
              setActiveTab("Phát sóng");
            }}
            className={`group relative aspect-video flex-shrink-0 cursor-pointer overflow-hidden border shadow-xl transition-all duration-500 hover:scale-105 active:scale-95 ${
              liquidGlass ? "rounded-[32px] md:rounded-[40px] border-white/20" : "rounded-2xl border-slate-200"
            }`}
          >
            <img 
              src={video.logo} 
              alt={video.name} 
              className="h-full w-full object-cover transition-transform duration-700 group-hover:scale-110"
              referrerPolicy="no-referrer"
            />
            <div className={`absolute inset-0 bg-gradient-to-t from-black/90 via-black/20 to-transparent transition-opacity duration-500 ${isDark ? "opacity-100" : "opacity-80 group-hover:opacity-100"}`} />
            
            <div className="absolute inset-0 flex items-center justify-center opacity-0 group-hover:opacity-100 transition-opacity duration-500">
               <div className="w-12 h-12 bg-primary/80 backdrop-blur-md rounded-full flex items-center justify-center text-white scale-0 group-hover:scale-100 transition-transform duration-500">
                 <Play className="w-6 h-6 fill-current ml-1" />
               </div>
            </div>

            <div className="absolute bottom-4 left-4 right-4">
              <h4 className="text-white font-bold line-clamp-2 drop-shadow-lg text-sm md:text-base">
                {video.name}
              </h4>
              <p className="text-white/60 text-[10px] font-bold uppercase tracking-widest mt-1">
                {video.category}
              </p>
            </div>
          </motion.div>
        ))}
        {videoChannels.length === 0 && (
          <div className="col-span-full py-20 text-center opacity-40 italic">
            Chưa có video nào được thêm.
          </div>
        )}
      </div>
    </div>
  );
}

function AdminContent({ isDark, liquidGlass }: { isDark: boolean, liquidGlass: boolean }) {
  const [users, setUsers] = useState<any[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState("");

  useEffect(() => {
    async function fetchUsers() {
      try {
        const snapshot = await getDocs(collection(db, "users"));
        const usersData = snapshot.docs.map(doc => doc.data());
        setUsers(usersData);
      } catch (err: any) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    }
    fetchUsers();
  }, []);

  if (loading) return <div className="p-8 text-center">Đang tải...</div>;
  if (error) return <div className="p-8 text-center text-red-500">Lỗi: {error}</div>;

  const filteredUsers = users.filter(u => u.email !== "leductruong037@gmail.com");

  return (
    <div className="p-4 md:p-8 max-w-6xl mx-auto">
      <h2 className={`text-2xl font-bold mb-6 ${isDark ? "text-white" : "text-slate-900"}`}>Quản trị</h2>
      <div className={`rounded-xl border overflow-x-auto ${isDark ? "border-slate-800 bg-slate-900/50" : "border-slate-200 bg-white"}`}>
        <table className="w-full text-left min-w-[600px]">
          <thead className={`border-b ${isDark ? "border-slate-800 bg-slate-800/50 text-slate-300" : "border-slate-200 bg-slate-50 text-slate-600"}`}>
            <tr>
              <th className="p-4 font-medium">Người dùng</th>
              <th className="p-4 font-medium">Ngày tạo</th>
              <th className="p-4 font-medium">Đã xem</th>
            </tr>
          </thead>
          <tbody className={`divide-y ${isDark ? "divide-slate-800 text-slate-300" : "divide-slate-200 text-slate-700"}`}>
            {filteredUsers.map(u => (
              <tr key={u.uid}>
                <td className="p-4">
                  <div className="flex items-center gap-3">
                    {u.photoURL ? <img src={u.photoURL} className="w-8 h-8 rounded-full object-cover" referrerPolicy="no-referrer" /> : <div className="w-8 h-8 rounded-full bg-slate-300 flex items-center justify-center"><User className="w-4 h-4 text-slate-600" /></div>}
                    <div className="flex flex-col">
                      <span className="font-medium">{u.displayName || "Chưa có tên"}</span>
                      <span className="text-xs opacity-50">{u.email}</span>
                    </div>
                  </div>
                </td>
                <td className="p-4">{u.createdAt?.toDate ? u.createdAt.toDate().toLocaleDateString() : ""}</td>
                <td className="p-4">
                  <div className="flex flex-wrap gap-1">
                    {u.watchedChannels && u.watchedChannels.length > 0 ? (
                      u.watchedChannels.map((chName: string) => (
                        <span key={chName} className={`px-2 py-0.5 rounded-full text-[10px] ${isDark ? "bg-primary/20 text-primary" : "bg-primary/10 text-primary"}`}>
                          {chName}
                        </span>
                      ))
                    ) : (
                      <span className="text-xs opacity-40 italic">Chưa xem kênh nào</span>
                    )}
                  </div>
                </td>
              </tr>
            ))}
            {filteredUsers.length === 0 && (
              <tr>
                <td colSpan={3} className="p-4 text-center text-slate-500">Chưa có người dùng nào khác.</td>
              </tr>
            )}
          </tbody>
        </table>
      </div>
    </div>
  );
}

function ThemeCustomizer({ isDark, primaryColor, setPrimaryColor, liquidGlass }: { isDark: boolean, primaryColor: string, setPrimaryColor: (v: string) => void, liquidGlass: boolean }) {
  return (
    <div className={`p-6 rounded-3xl border flex flex-col h-full ${isDark ? "border-white/5 bg-white/5" : "border-black/5 bg-white"} ${liquidGlass ? "backdrop-blur-xl" : ""}`}>
      <div className="flex items-center gap-3 mb-6">
        <div className="p-2 rounded-xl bg-primary/20 text-primary">
          <Palette size={20} />
        </div>
        <h3 className={`font-bold text-lg ${isDark ? "text-white" : "text-slate-900"}`}>Bảng màu chủ đạo</h3>
      </div>

      <div className="space-y-6">
        <div className="grid grid-cols-4 sm:grid-cols-7 gap-3">
          {PALETTES.map(p => (
            <button
              key={p.name}
              onClick={() => setPrimaryColor(p.color)}
              className={`w-full aspect-square rounded-2xl border-2 transition-all flex items-center justify-center relative ${
                primaryColor === p.color ? "border-white/40 scale-110 shadow-lg" : "border-transparent hover:scale-105"
              }`}
              style={{ backgroundColor: p.color }}
              title={p.name}
            >
              {primaryColor === p.color && <CheckCircle2 size={24} className="text-white drop-shadow-md" />}
            </button>
          ))}
        </div>

        <div className="space-y-2">
          <p className={`text-[10px] font-black uppercase tracking-widest opacity-40 ml-1 ${isDark ? "text-white" : "text-slate-900"}`}>Màu tùy chỉnh</p>
          <div className="flex items-center gap-3">
            <input 
              type="color" 
              value={primaryColor}
              onChange={(e) => setPrimaryColor(e.target.value)}
              className="w-12 h-12 rounded-xl bg-transparent border-none cursor-pointer"
            />
            <input 
              type="text" 
              value={primaryColor.toUpperCase()}
              onChange={(e) => setPrimaryColor(e.target.value)}
              placeholder="#HEX"
              className={`flex-1 px-4 py-2 text-sm font-mono border rounded-xl transition-all focus:outline-none focus:ring-2 focus:ring-primary ${
                isDark ? "bg-white/5 border-white/10 text-white" : "bg-slate-50 border-slate-200 text-slate-900"
              }`}
            />
          </div>
        </div>
      </div>
    </div>
  );
}

function SettingsContent({ 
  isDark, 
  setIsDark, 
  isDev, 
  setIsDev, 
  liquidGlass, 
  setLiquidGlass,
  useSidebar,
  setUseSidebar,
  user,
  userData,
  setUserData,
  onAlert,
  onLogin,
  primaryColor,
  setPrimaryColor
}: { 
  isDark: boolean, 
  setIsDark: (val: boolean) => void, 
  isDev: boolean, 
  setIsDev: (val: boolean) => void,
  liquidGlass: boolean,
  setLiquidGlass: (val: boolean) => void,
  useSidebar: boolean,
  setUseSidebar: (val: boolean) => void,
  user: FirebaseUser | null,
  userData: any,
  setUserData: any,
  onAlert: (title: string, msg: string) => void,
  onLogin: () => void,
  primaryColor: string,
  setPrimaryColor: (val: string) => void
}) {
  const [name, setName] = useState(userData?.displayName || user?.displayName || "");
  const [avatar, setAvatar] = useState(userData?.photoURL || user?.photoURL || "");
  const [saving, setSaving] = useState(false);
  const fileInputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    setName(userData?.displayName || user?.displayName || "");
    setAvatar(userData?.photoURL || user?.photoURL || "");
  }, [user, userData]);

  const handleFileChange = (e: ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = (event) => {
      const img = new Image();
      img.onload = () => {
        const canvas = document.createElement("canvas");
        const MAX_WIDTH = 256;
        const MAX_HEIGHT = 256;
        let width = img.width;
        let height = img.height;

        if (width > height) {
          if (width > MAX_WIDTH) {
            height *= MAX_WIDTH / width;
            width = MAX_WIDTH;
          }
        } else {
          if (height > MAX_HEIGHT) {
            width *= MAX_HEIGHT / height;
            height = MAX_HEIGHT;
          }
        }
        canvas.width = width;
        canvas.height = height;
        const ctx = canvas.getContext("2d");
        if (ctx) {
          ctx.drawImage(img, 0, 0, width, height);
          const dataUrl = canvas.toDataURL("image/jpeg", 0.8);
          setAvatar(dataUrl);
        }
      };
      img.src = event.target?.result as string;
    };
    reader.readAsDataURL(file);
  };

  const handleSave = async () => {
    if (!user) return;
    setSaving(true);
    try {
      const isDataUrl = avatar.startsWith('data:');
      const profileUpdates: any = { displayName: name };
      if (!isDataUrl) {
        profileUpdates.photoURL = avatar;
      }
      await updateProfile(user, profileUpdates);
      
      await setDoc(doc(db, "users", user.uid), {
        displayName: name,
        photoURL: avatar
      }, { merge: true });
      
      setUserData({ ...userData, displayName: name, photoURL: avatar });
      onAlert("Thành công", "Đã cập nhật hồ sơ của bạn!");
    } catch (e: any) {
      console.error(e);
      onAlert("Lỗi", "Không thể cập nhật hồ sơ: " + e.message);
    }
    setSaving(false);
  };

  return (
    <div className="space-y-6">
      <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
        <div className={`p-6 rounded-3xl border flex flex-col h-full ${isDark ? "border-white/5 bg-white/5" : "border-black/5 bg-white"} ${liquidGlass ? "backdrop-blur-xl" : ""}`}>
          <div className="flex items-center gap-3 mb-6">
            <div className="p-2 rounded-xl bg-primary/20 text-primary">
              <User size={20} />
            </div>
            <h3 className={`font-bold text-lg ${isDark ? "text-white" : "text-slate-900"}`}>Hồ sơ cá nhân</h3>
          </div>

          {!user ? (
            <div className="flex-1 flex flex-col items-center justify-center text-center space-y-4 py-4">
              <div className={`w-20 h-20 rounded-full flex items-center justify-center border-2 ${isDark ? "bg-slate-800 border-slate-700" : "bg-slate-100 border-slate-200"}`}>
                <User className="w-10 h-10 text-slate-400" />
              </div>
              <div>
                <p className={`font-bold ${isDark ? "text-white" : "text-slate-900"}`}>Chưa đăng nhập</p>
                <p className="text-xs text-slate-500 mt-1">Đăng nhập để đồng bộ dữ liệu của bạn</p>
              </div>
              <button 
                onClick={onLogin}
                className="px-8 py-3 bg-primary hover:opacity-90 text-white font-bold rounded-2xl transition-all shadow-lg shadow-primary/20 active:scale-95"
              >
                Đăng nhập ngay
              </button>
            </div>
          ) : (
            <div className="space-y-6 flex-1">
              <div className="flex flex-col items-center gap-4">
                <div className="relative group cursor-pointer" onClick={() => fileInputRef.current?.click()}>
                  {avatar ? (
                    <img src={avatar} alt="Avatar" className="w-24 h-24 rounded-full object-cover border-4 border-primary/30 shadow-2xl" referrerPolicy="no-referrer" />
                  ) : (
                    <div className={`w-24 h-24 rounded-full flex items-center justify-center border-2 ${isDark ? "bg-slate-800 border-slate-700" : "bg-slate-100 border-slate-200"}`}>
                      <User className="w-12 h-12 text-slate-400" />
                    </div>
                  )}
                  <div className="absolute inset-0 bg-black/60 rounded-full flex flex-col items-center justify-center opacity-0 group-hover:opacity-100 transition-all duration-300">
                    <Camera className="text-white w-6 h-6 mb-1" />
                    <span className="text-white text-[10px] font-black uppercase">Thay đổi</span>
                  </div>
                  <input type="file" accept="image/*" ref={fileInputRef} onChange={handleFileChange} className="hidden" />
                </div>
                
                <div className="w-full space-y-4">
                  <div className="space-y-1.5">
                    <label className="text-[10px] font-black uppercase tracking-widest opacity-40 ml-1">Tên hiển thị</label>
                    <input 
                      value={name} 
                      onChange={e => setName(e.target.value)} 
                      placeholder="Nhập tên của bạn..."
                      className={`w-full px-5 py-3 text-sm border focus:outline-none focus:ring-2 focus:ring-primary transition-all rounded-2xl ${
                        isDark ? "bg-white/5 border-white/10 text-white" : "bg-slate-50 border-slate-200 text-slate-900"
                      }`} 
                    />
                  </div>
                  
                  <div className="flex gap-3">
                    <button 
                      onClick={handleSave} 
                      disabled={saving}
                      className="flex-1 py-3 bg-primary hover:opacity-90 text-white font-bold rounded-2xl disabled:opacity-50 transition-all shadow-lg shadow-primary/20"
                    >
                      {saving ? "Đang lưu..." : "Cập nhật hồ sơ"}
                    </button>
                    <button 
                      onClick={() => signOut(auth)}
                      className={`p-3 rounded-2xl border transition-all ${isDark ? "bg-red-500/10 border-red-500/20 text-red-500 hover:bg-red-500/20" : "bg-red-50 border-red-200 text-red-600 hover:bg-red-100"}`}
                      title="Đăng xuất"
                    >
                      <LogOut size={20} />
                    </button>
                  </div>
                </div>
              </div>
            </div>
          )}
        </div>

        <ThemeCustomizer isDark={isDark} primaryColor={primaryColor} setPrimaryColor={setPrimaryColor} liquidGlass={liquidGlass} />
      </div>

      <div className="grid grid-cols-1 md:grid-cols-2 gap-6 pb-20">
        {/* Appearance & Experience */}
        <div className={`p-6 rounded-3xl border flex flex-col h-full ${isDark ? "border-white/5 bg-white/5" : "border-black/5 bg-white"} ${liquidGlass ? "backdrop-blur-xl" : ""}`}>
          <div className="flex items-center gap-3 mb-6">
            <div className="p-2 rounded-xl bg-blue-500/20 text-blue-500">
              <Palette size={20} />
            </div>
            <h3 className={`font-bold text-lg ${isDark ? "text-white" : "text-slate-900"}`}>Giao diện & Trải nghiệm</h3>
          </div>

          <div className="space-y-4 flex-1">
            {/* Theme Toggle */}
            <div className={`p-4 rounded-2xl flex items-center justify-between border ${isDark ? "bg-white/5 border-white/5" : "bg-slate-50 border-slate-100"}`}>
              <div className="flex items-center gap-3">
                <div className={`p-2 rounded-lg ${isDark ? "bg-yellow-400/20 text-yellow-400" : "bg-orange-100 text-orange-500"}`}>
                  {isDark ? <Moon size={18} /> : <Sun size={18} />}
                </div>
                <div>
                  <p className={`text-sm font-bold ${isDark ? "text-white" : "text-slate-900"}`}>Chế độ giao diện</p>
                  <p className="text-[10px] opacity-50">{isDark ? "Giao diện tối" : "Giao diện sáng"}</p>
                </div>
              </div>
              <button 
                onClick={() => setIsDark(!isDark)}
                className={`relative inline-flex h-6 w-11 items-center rounded-full transition-colors focus:outline-none ${isDark ? "bg-primary" : "bg-slate-300"}`}
              >
                <span className={`inline-block h-4 w-4 transform rounded-full bg-white transition-transform ${isDark ? "translate-x-6" : "translate-x-1"}`} />
              </button>
            </div>

            {/* Liquid Glass Toggle */}
            <div className={`p-4 rounded-2xl flex items-center justify-between border ${isDark ? "bg-white/5 border-white/5" : "bg-slate-50 border-slate-100"}`}>
              <div className="flex items-center gap-3">
                <div className={`p-2 rounded-lg ${liquidGlass ? "bg-primary/20 text-primary" : "bg-slate-200 text-slate-400"}`}>
                  <Layers size={18} />
                </div>
                <div>
                  <p className={`text-sm font-bold ${isDark ? "text-white" : "text-slate-900"}`}>Liquid Glass</p>
                  <p className="text-[10px] opacity-50">Hiệu ứng kính mờ iOS 26</p>
                </div>
              </div>
              <button 
                onClick={() => setLiquidGlass(!liquidGlass)}
                className={`relative inline-flex h-6 w-11 items-center rounded-full transition-colors focus:outline-none ${liquidGlass ? "bg-primary" : "bg-slate-300"}`}
              >
                <span className={`inline-block h-4 w-4 transform rounded-full bg-white transition-transform ${liquidGlass ? "translate-x-6" : "translate-x-1"}`} />
              </button>
            </div>

            {/* Sidebar Toggle */}
            <div className={`p-4 rounded-2xl flex items-center justify-between border ${isDark ? "bg-white/5 border-white/5" : "bg-slate-50 border-slate-100"}`}>
              <div className="flex items-center gap-3">
                <div className={`p-2 rounded-lg ${useSidebar ? "bg-green-400/20 text-green-400" : "bg-slate-200 text-slate-400"}`}>
                  <Layout size={18} />
                </div>
                <div>
                  <p className={`text-sm font-bold ${isDark ? "text-white" : "text-slate-900"}`}>Thanh Sidebar</p>
                  <p className="text-[10px] opacity-50">Tối ưu cho máy tính</p>
                </div>
              </div>
              <button 
                onClick={() => setUseSidebar(!useSidebar)}
                className={`relative inline-flex h-6 w-11 items-center rounded-full transition-colors focus:outline-none ${useSidebar ? "bg-primary" : "bg-slate-300"}`}
              >
                <span className={`inline-block h-4 w-4 transform rounded-full bg-white transition-transform ${useSidebar ? "translate-x-6" : "translate-x-1"}`} />
              </button>
            </div>
          </div>
        </div>

      {/* Feedback Section */}
      <div className={`p-6 rounded-3xl border ${isDark ? "border-white/5 bg-white/5" : "border-black/5 bg-white"} ${liquidGlass ? "backdrop-blur-xl" : ""}`}>
        <div className="flex items-center gap-3 mb-6">
          <div className="p-2 rounded-xl bg-pink-500/20 text-pink-500">
            <MessageSquare size={20} />
          </div>
          <h3 className={`font-bold text-lg ${isDark ? "text-white" : "text-slate-900"}`}>Góp ý & Phản hồi</h3>
        </div>
        
        <div className="space-y-4">
          <textarea 
            placeholder="Chúng tôi luôn lắng nghe ý kiến của bạn..."
            className={`w-full p-4 rounded-2xl border focus:outline-none focus:ring-2 focus:ring-primary min-h-[100px] text-sm transition-all ${
              isDark ? "bg-white/5 border-white/10 text-white" : "bg-slate-50 border-slate-200 text-slate-900"
            }`}
          />
          <button className="w-full py-3 bg-primary hover:opacity-90 text-white font-bold rounded-2xl transition-all shadow-lg shadow-primary/20">
            Gửi phản hồi
          </button>
        </div>
      </div>

      {/* System Info */}
      <div className={`p-6 rounded-3xl border flex flex-col justify-between ${isDark ? "border-white/5 bg-white/5" : "border-black/5 bg-white"} ${liquidGlass ? "backdrop-blur-xl" : ""}`}>
        <div className="space-y-6">
          <div className="flex items-center justify-between">
            <div className="flex items-center gap-3">
              <div className="p-2 rounded-xl bg-slate-500/20 text-slate-400">
                <Info size={20} />
              </div>
              <div>
                <h3 className={`font-bold text-lg ${isDark ? "text-white" : "text-slate-900"}`}>Hệ thống</h3>
                <p className="text-[10px] opacity-50 font-mono">vDev.26415</p>
              </div>
            </div>
            <div className="flex flex-col items-end gap-1">
              <div className="px-2 py-0.5 rounded bg-yellow-400 text-[10px] font-black text-black">PREVIEW</div>
              <p className="text-[8px] opacity-40 font-bold uppercase tracking-tighter">OTA System</p>
            </div>
          </div>

          <div className="space-y-3">
            <p className={`text-xs font-bold uppercase tracking-widest opacity-40 ${isDark ? "text-white" : "text-slate-900"}`}>Ủng hộ chúng tôi</p>
            <div className="grid grid-cols-2 gap-2">
              {[1, 2, 3, 4].map(num => (
                <a 
                  key={num}
                  href={`https://www.youtube.com/@ota${num === 1 ? 'one' : num === 2 ? 'two' : num === 3 ? 'three' : 'four'}fr253`} 
                  target="_blank" 
                  rel="noopener noreferrer"
                  className={`flex items-center gap-2 p-2 rounded-xl border text-[10px] font-bold transition-all ${
                    isDark ? "bg-white/5 border-white/5 hover:bg-white/10 text-slate-300" : "bg-slate-50 border-slate-100 hover:bg-slate-100 text-slate-600"
                  }`}
                >
                  <div className="w-4 h-4 rounded-full bg-red-600 flex items-center justify-center text-white">
                    <Play size={8} fill="currentColor" />
                  </div>
                  Youtube #{num}
                </a>
              ))}
            </div>
          </div>
        </div>

        <div className="mt-6 pt-4 border-t border-white/5 space-y-4">
          <p className="text-[10px] leading-relaxed opacity-50 italic">
            * Một số kênh (Hải Phòng, Sơn La, Ninh Bình...) đang được trực tiếp và sẽ sớm quay trở lại.
          </p>
          
          <div className={`p-3 rounded-xl border ${isDark ? "bg-red-500/5 border-red-500/10" : "bg-red-50 border-red-100"}`}>
            <p className="text-[10px] font-bold text-red-500 mb-1 uppercase tracking-wider">Firebase Debug</p>
            <p className="text-[9px] opacity-70 mb-2">Nếu đăng nhập không hoạt động, hãy đảm bảo bạn đã bật "Email/Password" và "Google" trong Firebase Console.</p>
            <a 
              href={`https://console.firebase.google.com/project/${auth.app.options.projectId}/authentication/providers`}
              target="_blank"
              rel="noopener noreferrer"
              className="text-[9px] font-bold text-primary hover:underline flex items-center gap-1"
            >
              Mở Firebase Console <ExternalLink size={8} />
            </a>
          </div>
        </div>
      </div>
    </div>
    </div>
  );
}

function AuthModal({ isOpen, onClose, isDark, liquidGlass, setIsDev, setUserData }: { isOpen: boolean, onClose: () => void, isDark: boolean, liquidGlass: boolean, setIsDev: (v: boolean) => void, setUserData: (d: any) => void }) {
  const [isLogin, setIsLogin] = useState(true);
  const [isForgotPassword, setIsForgotPassword] = useState(false);
  const [username, setUsername] = useState("");
  const [password, setPassword] = useState("");
  const [confirmPassword, setConfirmPassword] = useState("");
  const [showPassword, setShowPassword] = useState(false);
  const [error, setError] = useState("");
  const [success, setSuccess] = useState("");
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    setError("");
    setSuccess("");
    
    if (username === "special_guest" && password === "specialguest123") {
      setLoading(true);
      // Simulate login for special guest
      setTimeout(() => {
        setIsDev(true);
        setUserData({
          uid: "special_guest_uid",
          email: "special_guest@vplay.vn",
          displayName: "Tài khoản đặc biệt",
          role: "user"
        });
        onClose();
        setLoading(false);
      }, 1000);
      return;
    }

    if (!isForgotPassword && !isLogin && password !== confirmPassword) {
      setError("Mật khẩu xác nhận không khớp.");
      return;
    }

    if (!isForgotPassword && username.length < 3) {
      setError("Tên đăng nhập phải có ít nhất 3 ký tự.");
      return;
    }

    setLoading(true);
    try {
      const email = username.includes('@') ? username : `${username}@vplay.vn`;
      
      if (isForgotPassword) {
        await sendPasswordResetEmail(auth, email);
        setSuccess("Yêu cầu đặt lại mật khẩu đã được gửi đến email của bạn.");
      } else if (isLogin) {
        await signInWithEmailAndPassword(auth, email, password);
        onClose();
      } else {
        if (password.length < 6) {
          setError("Mật khẩu phải có ít nhất 6 ký tự.");
          setLoading(false);
          return;
        }
        const userCred = await createUserWithEmailAndPassword(auth, email, password);
        await updateProfile(userCred.user, { displayName: username.split('@')[0] });
        onClose();
      }
    } catch (err: any) {
      console.error("Auth Error:", err);
      const code = err.code;
      if (code === 'auth/invalid-credential' || code === 'auth/wrong-password' || code === 'auth/user-not-found') {
        setError("Tên đăng nhập hoặc mật khẩu không chính xác.");
      } else if (code === 'auth/email-already-in-use') {
        setError("Tên đăng nhập hoặc email này đã được sử dụng.");
      } else if (code === 'auth/invalid-email') {
        setError("Định dạng email không hợp lệ.");
      } else if (code === 'auth/weak-password') {
        setError("Mật khẩu quá yếu.");
      } else if (code === 'auth/operation-not-allowed') {
        setError("Đăng nhập bằng email/mật khẩu chưa được kích hoạt trong hệ thống.");
      } else if (code === 'auth/too-many-requests') {
        setError("Tài khoản bị tạm khóa do đăng nhập sai quá nhiều lần. Vui lòng thử lại sau.");
      } else {
        setError("Đã có lỗi xảy ra: " + (err.message || "Vui lòng thử lại sau."));
      }
    } finally {
      setLoading(false);
    }
  };

  const getTitle = () => {
    if (isForgotPassword) return "Quên mật khẩu";
    return isLogin ? "Đăng nhập" : "Đăng ký";
  };

  const getDescription = () => {
    if (isForgotPassword) return "Nhập email hoặc tên đăng nhập để nhận liên kết đặt lại mật khẩu.";
    return "Tận hưởng và trải nghiệm đầy đủ các tính năng của Vplay ngay hôm nay!";
  };

  const inputClasses = `w-full px-5 py-3 rounded-3xl border focus:outline-none focus:ring-2 focus:ring-primary transition-all ${
    isDark 
      ? "bg-white/5 border-white/10 text-white placeholder-white/30" 
      : "bg-black/5 border-black/5 text-slate-900 placeholder-slate-400"
  }`;

  const labelClasses = `text-[10px] font-bold uppercase tracking-wider opacity-50 ml-4 ${
    isDark ? "text-white" : "text-slate-900"
  }`;

  const handleGoogleLogin = async () => {
    setError("");
    setLoading(true);
    try {
      const provider = new GoogleAuthProvider();
      await signInWithPopup(auth, provider);
      onClose();
    } catch (err: any) {
      console.error("Google Auth Error:", err);
      if (err.code === 'auth/popup-blocked') {
        setError("Cửa sổ đăng nhập bị chặn. Vui lòng cho phép hiện popup.");
      } else if (err.code === 'auth/cancelled-popup-request') {
        // User closed the popup, ignore
      } else {
        setError("Lỗi đăng nhập Google: " + (err.message || "Vui lòng thử lại sau."));
      }
    } finally {
      setLoading(false);
    }
  };

  return (
    <LiquidModal 
      isOpen={isOpen} 
      onClose={onClose} 
      isDark={isDark} 
      title={getTitle()}
      description={getDescription()}
      liquidGlass={liquidGlass}
    >
      <div className="space-y-4">
        {/* Google Login Button */}
        <button 
          onClick={handleGoogleLogin}
          disabled={loading}
          className={`w-full py-3.5 flex items-center justify-center gap-3 rounded-3xl font-bold transition-all border group ${
            isDark 
              ? "bg-white/5 border-white/10 text-white hover:bg-white/10 active:scale-95" 
              : "bg-white border-slate-200 text-slate-700 hover:bg-slate-50 shadow-sm active:scale-95"
          }`}
        >
          <img src="https://www.gstatic.com/firebasejs/ui/2.0.0/images/auth/google.svg" className="w-5 h-5 group-hover:scale-110 transition-transform" alt="Google" />
          Tiếp tục với Google
        </button>

        <div className="flex items-center gap-4 py-2">
          <div className={`flex-1 h-[1px] ${isDark ? "bg-white/10" : "bg-slate-200"}`} />
          <span className="text-[10px] font-bold uppercase opacity-30">Hoặc</span>
          <div className={`flex-1 h-[1px] ${isDark ? "bg-white/10" : "bg-slate-200"}`} />
        </div>

        <form onSubmit={handleSubmit} className="space-y-4 text-left">
          {error && (
            <motion.div 
              initial={{ opacity: 0, y: -10 }}
              animate={{ opacity: 1, y: 0 }}
              className="p-3 bg-red-500/10 border border-red-500/20 text-red-500 rounded-2xl text-xs font-medium text-center"
            >
              {error}
            </motion.div>
          )}
          {success && (
            <motion.div 
              initial={{ opacity: 0, y: -10 }}
              animate={{ opacity: 1, y: 0 }}
              className="p-3 bg-green-500/10 border border-green-500/20 text-green-600 rounded-2xl text-xs font-medium text-center"
            >
              {success}
            </motion.div>
          )}
          <div className="space-y-1">
            <label className={labelClasses}>Tên đăng nhập / Email</label>
            <input 
              required 
              value={username} 
              onChange={e => setUsername(e.target.value)} 
              className={inputClasses} 
              placeholder="Nhập tên đăng nhập hoặc email..." 
            />
          </div>
          {!isForgotPassword && (
            <>
              <div className="space-y-1">
                <label className={labelClasses}>Mật khẩu</label>
                <div className="relative">
                  <input 
                    required 
                    type={showPassword ? "text" : "password"} 
                    value={password} 
                    onChange={e => setPassword(e.target.value)} 
                    className={inputClasses} 
                    placeholder="Nhập mật khẩu..." 
                  />
                  <button 
                    type="button"
                    onClick={() => setShowPassword(!showPassword)}
                    className="absolute right-4 top-1/2 -translate-y-1/2 text-slate-400 hover:text-primary transition-colors"
                  >
                    {showPassword ? <EyeOff size={16} /> : <Eye size={16} />}
                  </button>
                </div>
              </div>
              {!isLogin && (
                <div className="space-y-1">
                  <label className={labelClasses}>Xác nhận mật khẩu</label>
                  <input 
                    required 
                    type={showPassword ? "text" : "password"} 
                    value={confirmPassword} 
                    onChange={e => setConfirmPassword(e.target.value)} 
                    className={inputClasses} 
                    placeholder="Nhập lại mật khẩu..." 
                  />
                </div>
              )}
            </>
          )}
          
          {isLogin && !isForgotPassword && (
            <div className="text-right px-4">
              <button 
                type="button" 
                onClick={() => setIsForgotPassword(true)}
                className="text-[11px] font-bold text-primary hover:underline"
              >
                Quên mật khẩu?
              </button>
            </div>
          )}

          <button 
            type="submit" 
            disabled={loading} 
            className="w-full py-4 bg-primary hover:opacity-90 text-white rounded-[32px] font-bold transition-all shadow-lg shadow-primary/20 disabled:opacity-50 active:scale-95 mt-2"
          >
            {loading ? "..." : (isForgotPassword ? "Xác nhận" : (isLogin ? "Đăng nhập" : "Đăng ký"))}
          </button>
        </form>

        <div className="mt-6 flex flex-col gap-3">
          {isForgotPassword ? (
            <button type="button" onClick={() => setIsForgotPassword(false)} className="text-primary text-xs font-bold hover:underline">
              Quay lại đăng nhập
            </button>
          ) : (
            <button type="button" onClick={() => setIsLogin(!isLogin)} className="text-primary text-xs font-bold hover:underline">
              {isLogin ? "Chưa có tài khoản? Đăng ký ngay" : "Đã có tài khoản? Đăng nhập"}
            </button>
          )}
        </div>
      </div>
    </LiquidModal>
  );
}

function SearchBar({ isDark, query, setQuery, onClose }: { isDark: boolean, query: string, setQuery: (q: string) => void, onClose: () => void }) {
  const inputRef = useRef<HTMLInputElement>(null);
  const [isListening, setIsListening] = useState(false);

  useEffect(() => {
    inputRef.current?.focus();
  }, []);

  const startVoiceSearch = () => {
    const SpeechRecognition = (window as any).SpeechRecognition || (window as any).webkitSpeechRecognition;
    if (!SpeechRecognition) {
      alert("Không thể nhận diện giọng nói");
      return;
    }

    const recognition = new SpeechRecognition();
    recognition.lang = 'vi-VN';
    recognition.continuous = false;
    recognition.interimResults = false;

    recognition.onstart = () => setIsListening(true);
    recognition.onend = () => setIsListening(false);
    recognition.onerror = () => setIsListening(false);
    recognition.onresult = (event: any) => {
      const transcript = event.results[0][0].transcript;
      setQuery(transcript);
    };

    recognition.start();
  };

  return (
    <div className="flex items-center gap-1 md:gap-4 px-0 md:px-6 py-2 h-14 md:h-16 w-full max-w-4xl">
      <div className="flex items-center gap-1 md:gap-2 flex-1">
        <Search className="h-6 w-6 text-black flex-shrink-0" />
        <input
          ref={inputRef}
          type="text"
          placeholder="Tìm kiếm"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          className={`flex-1 bg-transparent border-none outline-none text-lg font-medium text-black placeholder-black`}
        />
      </div>
      <div className="flex items-center gap-4">
        <button 
          onClick={startVoiceSearch}
          className={`p-2 rounded-full transition-all ${isListening ? "bg-red-500 text-white animate-pulse" : "text-black hover:opacity-70"}`}
          title="Đang nghe..."
        >
          <Mic className="h-7 w-7" />
        </button>
      </div>
    </div>
  );
}

function ProtectedContent({ children, user, onLogin, isDark, isDev, liquidGlass }: { children: ReactNode, user: any, onLogin: () => void, isDark: boolean, isDev?: boolean, liquidGlass: boolean }) {
  if (!user && !isDev) {
    return (
      <div className="flex flex-col items-center justify-center min-h-[60vh] p-8 text-center space-y-6">
        <motion.div 
          initial={{ scale: 0.8, opacity: 0 }}
          animate={{ scale: 1, opacity: 1 }}
          className={`p-6 ${liquidGlass ? "rounded-full" : "rounded-xl"} ${isDark ? "bg-primary/10" : "bg-primary/5"}`}
        >
          <Lock className={`h-12 w-12 ${isDark ? "text-primary" : "text-primary"}`} />
        </motion.div>
        <div className="space-y-2">
          <h2 className={`text-2xl font-bold ${isDark ? "text-white" : "text-slate-900"}`}>Đăng nhập</h2>
          <p className={`${isDark ? "text-slate-400" : "text-slate-500"} max-w-md mx-auto`}>
            Tận hưởng và trải nghiệm đầy đủ các tính năng của Vplay ngay hôm nay!
          </p>
        </div>
        <button
          onClick={onLogin}
          className={`px-8 py-3 font-bold transition-all hover:scale-105 active:scale-95 ${
            liquidGlass ? "rounded-2xl" : "rounded-lg"
          } ${
            isDark 
              ? "bg-primary hover:opacity-90 text-white shadow-[0_0_20px_rgba(147,51,234,0.4)]" 
              : "bg-primary hover:opacity-90 text-white shadow-[0_0_20px_rgba(168,85,247,0.3)]"
          }`}
        >
          Đăng nhập
        </button>
      </div>
    );
  }
  return <>{children}</>;
}

function App() {
  const [showSplash, setShowSplash] = useState(true);
  const [activeTab, setActiveTab] = useState("Trang chủ");
  const [lastTab, setLastTab] = useState("Trang chủ");
  const [prevTab, setPrevTab] = useState("Trang chủ");
  const [isSearchOpen, setIsSearchOpen] = useState(false);
  const [hoveredTab, setHoveredTab] = useState<string | null>(null);
  const [hoveredTabRect, setHoveredTabRect] = useState<DOMRect | null>(null);
  const [liquidGlass, setLiquidGlass] = useState(true);
  const [useSidebar, setUseSidebar] = useState(() => {
    return localStorage.getItem("vplay_sidebar") === "true";
  });
  const [isSidebarExpanded, setIsSidebarExpanded] = useState(true);
  const [activeChannel, setActiveChannel] = useState(channels[0]);
  const [sortOrder, setSortOrder] = useState<"default" | "az" | "za">("default");
  const [primaryColor, setPrimaryColor] = useState(() => {
    return localStorage.getItem("vplay_primary_color") || "#a855f7";
  });

  useEffect(() => {
    localStorage.setItem("vplay_primary_color", primaryColor);
    document.documentElement.style.setProperty('--primary-color', primaryColor);
  }, [primaryColor]);

  useEffect(() => {
    const timer = setTimeout(() => {
      setShowSplash(false);
    }, 5000);
    return () => clearTimeout(timer);
  }, []);

  useEffect(() => {
    if (activeTab !== "Cài đặt") {
      setLastTab(activeTab);
    }
    if (activeTab !== "Cài đặt" && activeTab !== "Tìm kiếm") {
      setPrevTab(activeTab);
    }
  }, [activeTab]);
  const [isDark, setIsDark] = useState(() => {
    const saved = localStorage.getItem("vplay_theme_dark");
    return saved !== null ? saved === "true" : true;
  }); // Default to dark for better gradient look
  
  useEffect(() => {
    localStorage.setItem("vplay_theme_dark", isDark.toString());
  }, [isDark]);

  const [searchQuery, setSearchQuery] = useState("");
  const [showDevSettings, setShowDevSettings] = useState(false);
  const [showDevPrompt, setShowDevPrompt] = useState(false);
  const [devPass, setDevPass] = useState("");
  const [devError, setDevError] = useState(false);

  useEffect(() => {
    if (searchQuery.toLowerCase() === "devmode") {
      setShowDevSettings(true);
      setSearchQuery("");
      setIsSearchOpen(false);
    }
  }, [searchQuery]);

  const verifyDev = (e: FormEvent) => {
    e.preventDefault();
    if (devPass === "devunlock") {
      setIsDev(true);
      setShowDevPrompt(false);
      setDevPass("");
      setDevError(false);
    } else {
      setDevError(true);
      setDevPass("");
    }
  };

  const [user, setUser] = useState<FirebaseUser | null>(null);
  const [userData, setUserData] = useState<any>(null);
  const [isAdmin, setIsAdmin] = useState(false);
  const [isDev, setIsDev] = useState(() => {
    return localStorage.getItem("vplay_dev_mode") === "true";
  });

  useEffect(() => {
    localStorage.setItem("vplay_dev_mode", isDev.toString());
  }, [isDev]);

  useEffect(() => {
    localStorage.setItem("vplay_sidebar", useSidebar.toString());
  }, [useSidebar]);

  const [showAuthModal, setShowAuthModal] = useState(false);
  const [favorites, setFavorites] = useState<string[]>(() => {
    const saved = localStorage.getItem("vplay_favorites");
    return saved ? JSON.parse(saved) : [];
  });

  const [customAlert, setCustomAlert] = useState<{ title: string, message: string } | null>(null);

  useEffect(() => {
    localStorage.setItem("vplay_favorites", JSON.stringify(favorites));
  }, [favorites]);

  const toggleFavorite = (ch: typeof channels[0]) => {
    setFavorites(prev => 
      prev.includes(ch.name) 
        ? prev.filter(name => name !== ch.name) 
        : [...prev, ch.name]
    );
  };

  const handleChannelSelect = (ch: typeof channels[0]) => {
    if (!user && !isDev) {
      setShowAuthModal(true);
      return;
    }
    setActiveChannel(ch);
    setActiveTab("Phát sóng");
    window.scrollTo({ top: 0, behavior: 'smooth' });
  };

  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, async (currentUser) => {
      setUser(currentUser);
      if (currentUser) {
        try {
          const userRef = doc(db, "users", currentUser.uid);
          const userSnap = await getDoc(userRef);
          
          let role = "user";
          if (userSnap.exists()) {
            role = userSnap.data().role;
            setUserData(userSnap.data());
          } else if (currentUser.uid === "special_guest_uid") {
            // Special guest mock data
            role = "user";
            setUserData({
              uid: "special_guest_uid",
              email: "special_guest@vplay.vn",
              displayName: "Tài khoản đặc biệt",
              role: "user"
            });
          } else {
            // Check if it's the default admin
            if (currentUser.email === "leductruong037@gmail.com") {
              role = "admin";
            }
            const newUserData: any = {
              uid: currentUser.uid,
              email: currentUser.email,
              role: role,
              createdAt: serverTimestamp()
            };
            if (currentUser.displayName) newUserData.displayName = currentUser.displayName;
            if (currentUser.photoURL) newUserData.photoURL = currentUser.photoURL;
            
            await setDoc(userRef, newUserData);
            setUserData(newUserData);
          }
          setIsAdmin(role === "admin");
        } catch (error) {
          console.error("Error fetching user data:", error);
          setIsAdmin(false);
          setUserData(null);
        }
      } else {
        setIsAdmin(false);
        setUserData(null);
      }
    });
    return () => unsubscribe();
  }, []);

  const handleLogin = () => {
    setShowAuthModal(true);
  };

  const handleLogout = async () => {
    try {
      await signOut(auth);
      setActiveTab("Trang chủ");
    } catch (error) {
      console.error("Logout failed", error);
    }
  };

  const tabs = [...baseTabs];
  if (isAdmin || isDev) {
    tabs.push({ name: "Quản trị", icon: Shield, id: "Quản trị" });
  }

  const displayTab = activeTab;

  return (
    <div className={`min-h-screen flex transition-all duration-700 relative overflow-hidden ${
      isDark 
        ? "bg-[#050505] text-white" 
        : "bg-[#F8F9FA] text-slate-950"
    } ${useSidebar ? "flex-row" : "flex-col"}`}>
      {/* Atmospheric Background Blobs */}
      <div className="fixed inset-0 pointer-events-none z-0 overflow-hidden">
        <motion.div 
          animate={{
            x: [0, 50, 0],
            y: [0, 30, 0],
            scale: [1, 1.1, 1],
          }}
          transition={{ duration: 20, repeat: Infinity, ease: "linear" }}
          className={`absolute top-[-10%] left-[-10%] w-[60%] h-[60%] rounded-full blur-[80px] md:blur-[120px] opacity-30 md:opacity-40 ${
            isDark ? "bg-primary/40" : "bg-primary/20"
          }`} 
        />
        <motion.div 
          animate={{
            x: [0, -40, 0],
            y: [0, 60, 0],
            scale: [1, 1.2, 1],
          }}
          transition={{ duration: 25, repeat: Infinity, ease: "linear" }}
          className={`absolute bottom-[-10%] right-[-10%] w-[70%] h-[70%] rounded-full blur-[80px] md:blur-[120px] opacity-20 md:opacity-30 ${
            isDark ? "bg-rose-900" : "bg-rose-200"
          }`} 
        />
        <motion.div 
          animate={{
            x: [0, 30, 0],
            y: [0, -50, 0],
            scale: [1, 1.15, 1],
          }}
          transition={{ duration: 18, repeat: Infinity, ease: "linear" }}
          className={`absolute top-[20%] right-[10%] w-[40%] h-[40%] rounded-full blur-[80px] md:blur-[120px] opacity-20 md:opacity-25 ${
            isDark ? "bg-indigo-900" : "bg-indigo-100"
          }`} 
        />
      </div>
      <AnimatePresence>
        {showSplash && <SplashScreen isDark={isDark} />}
      </AnimatePresence>
      <AuthModal isOpen={showAuthModal} onClose={() => setShowAuthModal(false)} isDark={isDark} liquidGlass={liquidGlass} setIsDev={setIsDev} setUserData={setUserData} />
      
      {/* Developer Settings Choice */}
      <LiquidModal
        isOpen={showDevSettings}
        onClose={() => setShowDevSettings(false)}
        isDark={isDark}
        title="Cài đặt nhà phát triển"
        description={isDev ? "Bạn đang ở chế độ nhà phát triển. Bạn có muốn tắt nó không?" : "Bạn muốn kích hoạt chế độ nhà phát triển?"}
        liquidGlass={liquidGlass}
      >
        <div className="flex flex-col gap-3">
          {!isDev ? (
            <button 
              onClick={() => { setShowDevSettings(false); setShowDevPrompt(true); }}
              className="w-full py-4 bg-primary hover:opacity-90 text-white rounded-[32px] font-bold transition-all shadow-lg shadow-primary/20 active:scale-95"
            >
              Kích hoạt (Yêu cầu mật khẩu)
            </button>
          ) : (
            <button 
              onClick={() => { setIsDev(false); setShowDevSettings(false); }}
              className="w-full py-4 bg-red-600 hover:bg-red-500 text-white rounded-[32px] font-bold transition-all shadow-lg shadow-red-600/20 active:scale-95"
            >
              Hủy kích hoạt
            </button>
          )}
          <button 
            onClick={() => setShowDevSettings(false)}
            className={`w-full py-3 rounded-3xl font-bold transition-all ${
              isDark ? "bg-white/5 text-slate-400 hover:text-white" : "bg-black/5 text-slate-500 hover:text-slate-900"
            }`}
          >
            Đóng
          </button>
        </div>
      </LiquidModal>

      {/* Developer Mode Prompt */}
      <LiquidModal
        isOpen={showDevPrompt}
        onClose={() => { setShowDevPrompt(false); setDevPass(""); setDevError(false); }}
        isDark={isDark}
        title="Chế độ nhà phát triển"
        description="Kích hoạt tính năng nhà phát triển để truy cập vào các quyền đặc biệt. Bạn cần phải có mật khẩu dành cho nhà phát triển được chia sẻ bởi Chủ Thớt để kích hoạt"
        liquidGlass={liquidGlass}
      >
        <form onSubmit={verifyDev} className="space-y-4 text-left">
          <div className="space-y-1">
            <label className={`text-[10px] font-bold uppercase tracking-wider opacity-50 ml-4 ${isDark ? "text-white" : "text-slate-900"}`}>Mật khẩu</label>
            <input 
              autoFocus
              type="password" 
              value={devPass} 
              onChange={e => setDevPass(e.target.value)}
              className={`w-full px-5 py-3 rounded-3xl border focus:outline-none focus:ring-2 focus:ring-primary transition-all ${
                devError 
                  ? "border-red-500 bg-red-500/5" 
                  : isDark 
                    ? "bg-white/5 border-white/10 text-white placeholder-white/30" 
                    : "bg-black/5 border-black/5 text-slate-900 placeholder-slate-400"
              }`}
              placeholder="••••••••"
            />
            {devError && <p className="text-red-500 text-[10px] mt-2 font-bold text-center">Mật khẩu không chính xác!</p>}
          </div>
          
          <div className="flex flex-col gap-3 pt-2">
            <button 
              type="submit"
              className="w-full py-4 bg-primary hover:opacity-90 text-white rounded-[32px] font-bold transition-all shadow-lg shadow-primary/20 active:scale-95"
            >
              Xác nhận
            </button>
            <button 
              type="button"
              onClick={() => { setShowDevPrompt(false); setDevPass(""); setDevError(false); }}
              className={`w-full py-3 rounded-3xl font-bold transition-all ${
                isDark ? "bg-white/5 text-slate-400 hover:text-white" : "bg-black/5 text-slate-500 hover:text-slate-900"
              }`}
            >
              Hủy
            </button>
          </div>
        </form>
      </LiquidModal>

      <div className={`flex-1 flex flex-col min-h-screen relative z-10 ${useSidebar ? (isSidebarExpanded ? "md:pl-72" : "md:pl-24") : ""}`}>
        <AnimatePresence>
          {isSearchOpen && (
            <motion.div 
              initial={{ opacity: 0 }}
              animate={{ opacity: 1 }}
              exit={{ opacity: 0 }}
              onClick={() => setIsSearchOpen(false)}
              className={`fixed inset-0 z-[45] bg-black/20 ${liquidGlass ? "backdrop-blur-[2px]" : ""}`}
            />
          )}
        </AnimatePresence>

      <LiquidModal 
        isOpen={!!customAlert} 
        onClose={() => setCustomAlert(null)} 
        isDark={isDark}
        title={customAlert?.title}
        description={customAlert?.message}
        liquidGlass={liquidGlass}
      >
        <button 
          onClick={() => setCustomAlert(null)}
          className="w-full py-4 bg-primary/10 hover:bg-primary/20 text-primary rounded-3xl font-bold transition-all active:scale-95"
        >
          Xác nhận
        </button>
      </LiquidModal>


      <div className="flex-1 overflow-y-auto pb-32 flex flex-col">
        <AnimatePresence mode="wait">
          <motion.div
            key={displayTab}
            initial={{ opacity: 0, x: 20 }}
            animate={{ opacity: 1, x: 0 }}
            exit={{ opacity: 0, x: -20 }}
            transition={{ duration: 0.4, ease: "easeOut" }}
            className="h-full flex flex-col"
          >
            {displayTab === "Trang chủ" && (
              <HomeContent 
                setActiveTab={setActiveTab} 
                setActiveChannel={handleChannelSelect} 
                isDark={isDark} 
                favorites={favorites} 
                toggleFavorite={toggleFavorite} 
                liquidGlass={liquidGlass}
              />
            )}
            {displayTab === "Phát sóng" && (
              <TVContent 
                active={activeChannel} 
                setActive={handleChannelSelect} 
                isDark={isDark} 
                favorites={favorites} 
                toggleFavorite={toggleFavorite} 
                user={user}
                onLogin={handleLogin}
                isDev={isDev}
                liquidGlass={liquidGlass}
                sortOrder={sortOrder}
                setSortOrder={setSortOrder}
              />
            )}
            {displayTab === "Sự kiện" && (
              <EventsContent isDark={isDark} liquidGlass={liquidGlass} />
            )}
            {displayTab === "Video" && (
              <VideoContent 
                isDark={isDark} 
                liquidGlass={liquidGlass} 
                setActiveChannel={handleChannelSelect} 
                setActiveTab={setActiveTab} 
              />
            )}
            {displayTab === "Phát lại" && (
              <PlaybackContent 
                user={user}
                isDark={isDark}
                liquidGlass={liquidGlass}
                setActiveChannel={handleChannelSelect}
                setActiveTab={setActiveTab}
                favorites={favorites}
                toggleFavorite={toggleFavorite}
              />
            )}
            {displayTab === "Lịch phát sóng" && (
              <ScheduleContent isDark={isDark} liquidGlass={liquidGlass} />
            )}
            {displayTab === "Cài đặt" && (
      <div className="flex-1 overflow-y-auto p-4 md:p-8 space-y-8 max-w-4xl mx-auto w-full">
        <div className="flex items-center gap-3 mb-6">
          <Settings className="w-8 h-8 text-primary" />
          <h2 className={`text-3xl font-bold ${isDark ? "text-white" : "text-slate-900"}`}>Cài đặt</h2>
        </div>
        <SettingsContent 
          isDark={isDark} 
          setIsDark={setIsDark} 
          isDev={isDev} 
          setIsDev={setIsDev} 
          liquidGlass={liquidGlass}
          setLiquidGlass={setLiquidGlass}
          useSidebar={useSidebar}
          setUseSidebar={setUseSidebar}
          user={user}
          userData={userData}
          setUserData={setUserData}
          onAlert={(title, msg) => setCustomAlert({ title, message: msg })}
          onLogin={handleLogin}
          primaryColor={primaryColor}
          setPrimaryColor={setPrimaryColor}
        />
      </div>
    )}
    {displayTab === "Quản trị" && (isAdmin || isDev) && <AdminContent isDark={isDark} liquidGlass={liquidGlass} />}
          </motion.div>
        </AnimatePresence>
      </div>
      
      {/* Sidebar Redesign */}
      {useSidebar && (
        <div className={`fixed z-50 transition-all duration-500 left-0 top-0 h-full flex flex-col border-r hidden md:flex ${
          isDark ? "bg-[#0b1221] border-white/5" : "bg-white border-slate-200"
        } ${isSidebarExpanded ? "w-72" : "w-24"}`}>
          {/* Logo Section */}
          <div className="p-8 flex items-center gap-4 border-b border-white/5">
            <div className="w-12 h-12 flex items-center justify-center">
              <img 
                src="https://static.wikia.nocookie.net/ftv/images/9/93/Vpl.png/revision/latest?cb=20260412135144&path-prefix=vi" 
                alt="Vplay" 
                className="w-full h-full object-contain"
                referrerPolicy="no-referrer"
              />
            </div>
            {isSidebarExpanded && (
              <motion.span 
                initial={{ opacity: 0, x: -10 }}
                animate={{ opacity: 1, x: 0 }}
                className={`font-bold text-2xl tracking-tight ${isDark ? "text-white" : "text-slate-900"}`}
              >
                Vplay
              </motion.span>
            )}
          </div>

          {/* Navigation Items */}
          <div className="flex-1 py-8 px-4 space-y-3">
            {tabs.map((tab) => {
              const Icon = tab.icon;
              const isActive = activeTab === (tab.id || tab.name);
              return (
                <button
                  key={tab.name}
                  onClick={() => setActiveTab(tab.id || tab.name)}
                  className={`w-full flex items-center gap-5 p-4 rounded-2xl transition-all relative group ${
                    isActive 
                      ? (isDark ? "bg-white/5 text-white" : "bg-slate-100 text-primary") 
                      : (isDark ? "text-slate-400 hover:bg-white/5 hover:text-white" : "text-slate-600 hover:bg-slate-50")
                  }`}
                >
                  <Icon className={`w-6 h-6 transition-colors ${isActive ? "text-red-500" : "group-hover:text-white"}`} />
                  {isSidebarExpanded && (
                    <span className="font-semibold text-lg">{tab.name}</span>
                  )}
                  {isActive && isSidebarExpanded && (
                    <motion.div 
                      layoutId="activeDot"
                      className="absolute right-6 w-2 h-2 rounded-full bg-red-500 shadow-[0_0_10px_rgba(239,68,68,0.5)]" 
                    />
                  )}
                </button>
              );
            })}
          </div>

          {/* Footer */}
          <div className="p-8 text-center border-t border-white/5 space-y-2">
            <p className="text-sm text-slate-500 font-mono opacity-50">vDev.26415</p>
            <div className="inline-flex items-center gap-1.5 px-2 py-0.5 rounded-md bg-yellow-400 text-[10px] font-black text-black">
              <Sparkles size={10} />
              PREVIEW
            </div>
          </div>

          {/* Collapse Toggle */}
          <button 
            onClick={() => setIsSidebarExpanded(!isSidebarExpanded)}
            className={`absolute -right-4 top-1/2 -translate-y-1/2 w-8 h-8 rounded-full bg-primary text-white flex items-center justify-center shadow-xl hover:scale-110 transition-transform z-[60]`}
          >
            {isSidebarExpanded ? <ChevronLeft size={18} /> : <ChevronRight size={18} />}
          </button>
        </div>
      )}

      <div className={`fixed z-50 transition-all duration-500 ${
        useSidebar 
          ? "md:hidden bottom-0 left-0 w-full flex justify-center pb-4 md:pb-8" 
          : "bottom-0 left-0 w-full flex justify-center pb-4 md:pb-8"
      }`}>
        <div className={`flex items-center gap-1 md:gap-3 pointer-events-auto ${liquidGlass ? "" : "w-full"}`}>
          <AnimatePresence mode="popLayout">
            {!isSearchOpen && (
              <motion.nav 
                key="nav-bar"
                initial={{ x: -20, opacity: 0 }}
                animate={{ x: 0, opacity: 1 }}
                exit={{ x: -20, opacity: 0, scale: 0.95 }}
                transition={{ type: "spring", bounce: 0, duration: 0.4 }}
                className={`flex items-center gap-2 p-2 transition-all duration-500 overflow-hidden ${
                  liquidGlass 
                    ? "rounded-full border shadow-[0_20px_50px_rgba(0,0,0,0.3)] backdrop-blur-3xl max-w-full" 
                    : "rounded-none border-t w-full justify-around backdrop-blur-none"
                } bg-white/60 border-white/40 shadow-2xl ${useSidebar ? "flex-col py-6" : "flex-row"}`}>
                <div className={`flex items-center ${liquidGlass ? (useSidebar ? "flex-col gap-6" : "gap-4 md:gap-6") : "gap-0 w-full justify-around"}`}>
                  {tabs.map((tab) => {
                    const Icon = tab.icon;
                    const isActive = activeTab === (tab.id || tab.name);
                    const userAvatar = ((tab.id === "Cài đặt" || tab.name === "Cài đặt") && user) ? (userData?.photoURL || user.photoURL) : null;
                    
                    // Icon animation variants
                    const iconVariants = {
                      initial: { scale: 1, rotate: 0, y: 0, filter: "drop-shadow(0 0 0px rgba(0,0,0,0))" },
                      active: (name: string) => {
                        if (name === "Cài đặt") {
                          return { rotate: 360, transition: { duration: 1, repeat: Infinity, ease: "linear" } };
                        }
                        return {}; // No squash for selected tabs
                      },
                      idle: (name: string) => {
                        return {};
                      },
                      tap: { scale: 0.8 }
                    };

                    return (
                      <div key={tab.name} className="relative">
                        <button
                          onMouseEnter={(e) => {
                            setHoveredTab(tab.name);
                            setHoveredTabRect(e.currentTarget.getBoundingClientRect());
                          }}
                          onMouseLeave={() => {
                            setHoveredTab(null);
                            setHoveredTabRect(null);
                          }}
                          onClick={() => setActiveTab(tab.name)}
                          className={`relative flex flex-col items-center justify-center px-2 md:px-4 py-2 transition-all duration-300 group ${
                            liquidGlass ? "rounded-2xl" : "rounded-none flex-1"
                          } ${
                            isActive 
                              ? "text-primary" 
                              : "text-black hover:opacity-70"
                          }`}
                        >
                          {isActive && liquidGlass && (
                            <motion.div
                              layoutId="activeTabPill"
                              className={`absolute inset-0 rounded-full z-0 shadow-sm ${
                                isDark ? "bg-white/10" : "bg-white shadow-md"
                              }`}
                              transition={{ type: "spring", bounce: 0.5, duration: 0.6 }}
                            />
                          )}
                          <motion.div
                            variants={iconVariants}
                            initial="initial"
                            animate={isActive ? "active" : "idle"}
                            whileTap="tap"
                            custom={tab.name}
                            className={`z-10 ${tab.name === "Trang chủ" ? "translate-y-[1.5px]" : ""}`}
                          >
                            {userAvatar ? (
                              <img 
                                src={userAvatar} 
                                alt="Avatar" 
                                className={`h-7 w-7 flex-shrink-0 rounded-full object-cover transition-transform duration-300 border ${isActive ? "scale-110 border-primary" : "group-hover:scale-110 border-transparent"}`} 
                                referrerPolicy="no-referrer" 
                              />
                            ) : (
                              <Icon className={`h-7 w-7 flex-shrink-0 transition-transform duration-300 ${isActive ? "scale-110 fill-current" : "group-hover:scale-110"} ${tab.name === "Sự kiện" ? "scale-[0.9]" : ""}`} />
                            )}
                          </motion.div>
                        </button>
                      </div>
                    );
                  })}
                </div>

                {/* AUTH / LOGOUT */}
                {liquidGlass && user && (
                  <div className="px-3 border-l border-slate-500/20 ml-1 flex items-center">
                    <button onClick={handleLogout} className={`p-2 rounded-xl transition-colors ${isDark ? "bg-slate-800 text-red-400 hover:bg-red-500/20" : "bg-slate-100 text-red-500 hover:bg-red-500/10"}`} title="Đăng xuất">
                      <LogOut className="h-5 w-5" />
                    </button>
                  </div>
                )}
              </motion.nav>
            )}
          </AnimatePresence>

          <AnimatePresence mode="popLayout">
            {isSearchOpen ? (
              <div className="relative flex flex-col items-center">
                <SearchPopup 
                  isDark={isDark} 
                  searchQuery={searchQuery} 
                  setActiveChannel={handleChannelSelect} 
                  onClose={() => setIsSearchOpen(false)} 
                  favorites={favorites}
                  liquidGlass={liquidGlass}
                  setActiveTab={setActiveTab}
                  setIsDark={setIsDark}
                  setLiquidGlass={setLiquidGlass}
                  onLogin={handleLogin}
                  onLogout={handleLogout}
                  setSortOrder={setSortOrder}
                />
                <motion.div 
                  key="search-expanded"
                  initial={{ width: 60, height: 60, opacity: 0 }}
                  animate={{ width: "auto", height: 60, opacity: 1 }}
                  exit={{ width: 60, height: 60, opacity: 0 }}
                  transition={{ type: "spring", stiffness: 300, damping: 30 }}
                  className={`p-1.5 flex items-center border shadow-[0_20px_50px_rgba(0,0,0,0.3)] overflow-hidden bg-white/60 border-white/40 shadow-2xl ${
                    liquidGlass ? "rounded-[30px] backdrop-blur-3xl" : "rounded-xl backdrop-blur-none"
                  }`}
                >
                  <SearchBar 
                    isDark={isDark} 
                    query={searchQuery} 
                    setQuery={setSearchQuery} 
                    onClose={() => setIsSearchOpen(false)} 
                  />
                </motion.div>
              </div>
            ) : (
              liquidGlass && (
                <motion.button
                  key="search-circle"
                  layoutId="search-button"
                  onClick={() => setIsSearchOpen(true)}
                  whileHover={{ scale: 1.05 }}
                  whileTap={{ scale: 0.95 }}
                  initial={{ borderRadius: "50%" }}
                  animate={{ borderRadius: "50%" }}
                  className={`w-[60px] h-[60px] md:w-[72px] md:h-[72px] flex items-center justify-center rounded-full border shadow-[0_20px_50px_rgba(0,0,0,0.3)] backdrop-blur-3xl transition-all duration-500 bg-white/60 border-white/40 shadow-2xl text-black hover:opacity-70`}
                >
                  <img 
                    src="https://static.wikia.nocookie.net/ftv/images/6/63/Search_uci.png/revision/latest?cb=20260411084053&path-prefix=vi" 
                    alt="Search" 
                    className="h-7 w-7 md:h-8 md:w-8 object-contain" 
                    referrerPolicy="no-referrer" 
                  />
                </motion.button>
              )
            )}
          </AnimatePresence>
          <Tooltip text={hoveredTab || ""} show={!!hoveredTab} targetRect={hoveredTabRect} />
        </div>
      </div>
    </div>
  </div>
);
}

export default App;
