import { useState, useEffect } from "react";
import { Card, CardContent } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Tabs, TabsList, TabsTrigger, TabsContent } from "@/components/ui/tabs";
import { initializeApp } from "firebase/app";
import { getFirestore, collection, addDoc, getDocs, query, where } from "firebase/firestore";
import { getAuth, signInWithPopup, GoogleAuthProvider, signOut, onAuthStateChanged } from "firebase/auth";

const firebaseConfig = {
  apiKey: "AIzaSyACSCO_KtdQoT-wisKku1UKHF3PnEGZUlI",
  authDomain: "talaa-48b1e.firebaseapp.com",
  projectId: "talaa-48b1e",
  storageBucket: "talaa-48b1e.appspot.com",
  messagingSenderId: "526906360747",
  appId: "1:526906360747:web:b1601e36528f4fd34748ee"
};

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
const auth = getAuth(app);
const provider = new GoogleAuthProvider();

export default function TalaaApp() {
  const [language, setLanguage] = useState("ru");
  const [region, setRegion] = useState("");
  const [crop, setCrop] = useState("");
  const [user, setUser] = useState(null);
  const [history, setHistory] = useState([]);
  const [weather, setWeather] = useState(null);

  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
      setUser(currentUser);
      if (currentUser) fetchHistory(currentUser.uid);
    });
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    const fetchCoordinatesAndWeather = async () => {
      try {
        const response = await fetch(`https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(region + ", Кыргызстан")}`);
        const data = await response.json();
        if (data && data.length > 0) {
          const { lat, lon } = data[0];
          const weatherRes = await fetch(`https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current_weather=true`);
          const weatherData = await weatherRes.json();
          setWeather(weatherData.current_weather);
        } else {
          console.error("Регион не найден");
          setWeather(null);
        }
      } catch (err) {
        console.error("Ошибка получения координат или погоды:", err);
        setWeather(null);
      }
    };
    if (region) fetchCoordinatesAndWeather();
  }, [region]);

  const handleLogin = async () => {
    try {
      await signInWithPopup(auth, provider);
    } catch (e) {
      console.error("Ошибка входа:", e);
    }
  };

  const handleLogout = async () => {
    try {
      await signOut(auth);
      setHistory([]);
    } catch (e) {
      console.error("Ошибка выхода:", e);
    }
  };

  const handleSave = async () => {
    if (!user) {
      alert("Сначала войдите в аккаунт");
      return;
    }
    try {
      const docRef = await addDoc(collection(db, "farmerInputs"), {
        uid: user.uid,
        email: user.email,
        region,
        crop,
        createdAt: new Date().toISOString(),
      });
      alert("Данные успешно сохранены в Firebase!");
      fetchHistory(user.uid);
    } catch (e) {
      console.error("Ошибка при сохранении данных:", e);
      alert("Ошибка при сохранении данных");
    }
  };

  const fetchHistory = async (uid) => {
    const q = query(collection(db, "farmerInputs"), where("uid", "==", uid));
    const querySnapshot = await getDocs(q);
    const data = querySnapshot.docs.map((doc) => doc.data());
    setHistory(data);
  };

  return (
    <div className="p-4 md:p-6 space-y-6 max-w-md mx-auto">
      <h1 className="text-2xl md:text-3xl font-bold text-center">Talaa App</h1>

      {user ? (
        <div className="text-center text-sm">
          <p>Вы вошли как: <strong>{user.email}</strong></p>
          <Button variant="ghost" onClick={handleLogout}>Выйти</Button>
        </div>
      ) : (
        <div className="text-center">
          <Button onClick={handleLogin}>Войти через Google</Button>
        </div>
      )}

      <Tabs defaultValue="home">
        <TabsList className="grid grid-cols-2 sm:grid-cols-4 gap-2">
          <TabsTrigger value="home">Главная</TabsTrigger>
          <TabsTrigger value="weather">Погода</TabsTrigger>
          <TabsTrigger value="market">Маркет</TabsTrigger>
          <TabsTrigger value="chat">Чат</TabsTrigger>
        </TabsList>

        <TabsContent value="home">
          <Card>
            <CardContent className="space-y-4 p-4">
              <p>Добро пожаловать в Talaa — ваш цифровой помощник фермера!</p>
              <Input
                placeholder="Ваш регион (например: Чуйская область)"
                value={region}
                onChange={(e) => setRegion(e.target.value)}
              />
              <Input
                placeholder="Основная культура (напр. клевер, пшеница)"
                value={crop}
                onChange={(e) => setCrop(e.target.value)}
              />
              <Button onClick={handleSave} className="w-full">Сохранить</Button>
              {history.length > 0 && (
                <div className="mt-4 space-y-2">
                  <h2 className="font-semibold">История записей:</h2>
                  {history.map((entry, i) => (
                    <div key={i} className="text-sm border-b py-1">
                      {entry.region} — {entry.crop} ({new Date(entry.createdAt).toLocaleString()})
                    </div>
                  ))}
                </div>
              )}
            </CardContent>
          </Card>
        </TabsContent>

        <TabsContent value="weather">
          <Card>
            <CardContent className="p-4 space-y-2">
              <h2 className="font-semibold text-lg">Погода</h2>
              {weather ? (
                <div>
                  <p>Температура: {weather.temperature}°C</p>
                  <p>Скорость ветра: {weather.windspeed} км/ч</p>
                  <p>Время: {new Date(weather.time).toLocaleString()}</p>
                </div>
              ) : (
                <p>Введите корректный регион для получения прогноза.</p>
              )}
            </CardContent>
          </Card>
        </TabsContent>

        <TabsContent value="market">
          <Card>
            <CardContent className="p-4">
              <p>Маркетплейс скоро будет доступен.</p>
            </CardContent>
          </Card>
        </TabsContent>

        <TabsContent value="chat">
          <Card>
            <CardContent className="p-4">
              <p>Чат сообщества Talaa скоро появится.</p>
            </CardContent>
          </Card>
        </TabsContent>
      </Tabs>

      <div className="text-center text-sm">
        Язык интерфейса:
        <div className="flex justify-center space-x-2 mt-2">
          <Button variant="ghost" onClick={() => setLanguage("ru")}>Русский</Button>
          <Button variant="ghost" onClick={() => setLanguage("kg")}>Кыргызча</Button>
        </div>
      </div>
    </div>
  );
}
