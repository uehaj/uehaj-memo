---
date: 2020-02-20T15:08:12.174Z
path: /60abb9be0dfac0755dcf.md/index.html
template: BlogPost
title: 第13回オフラインリアルタイムどう書くの問題をFregeで解く
tags: Frege doukaku
author: uehaj
slide: false
---
[第13回 オフラインリアルタイムどう書く](http://atnd.org/events/41603)の問題「[積み木の水槽](http://nabetani.sakura.ne.jp/hena/ord13blocktup/)」を、JVM上で動作するHaskellライクな言語[Frege](https://github.com/Frege/frege/wiki/_pages)(フレーゲ)で解きました。

```frege:shortest.fr
-- http://nabetani.sakura.ne.jp/hena/ord13blocktup/
     
data Cell = Wall | Empty | Water
derive Eq Cell -- Haskellのdata Cell = ... deriving(Eq)

-- セルを表示する
instance Show Cell where
    show Wall = "*"
    show Empty = " "
    show Water = "+"

-- 盤面を表示する
data Matrix = Matrix [ [ Cell ] ] Int Int
derive Eq Matrix
instance Show Matrix where
    show (Matrix xs w h) = "w="++show w++",h="++show h++"\n"++ unlines (map showLine xs)
        where
          showLine :: [Cell] -> String
          showLine line = "[" ++ foldl (++) "" (map show line) ++ "]"

-- 盤面を生成する
toMatrix :: String -> Matrix
toMatrix str =
    let maxx = length str
        maxy = maximum $ map (\ch -> ord ch - ord '0') (unpacked str)
        makeLine n maxn = replicate n Wall ++ replicate (maxn-n) Empty
    in Matrix (map (\ch -> makeLine (ord ch - ord '0') maxy) (unpacked str)) maxx maxy

-- 指定した座標(xPos,yPos)のセル内容を取得する
getCell :: Matrix -> Int -> Int -> Cell
getCell (Matrix mat w h) xPos yPos
      | (0 <= xPos) && ( xPos < w ) && (0 <= yPos) && (yPos < h) = mat !! xPos !! yPos
      | otherwise = Empty

-- Matrix型の盤面の指定した座標(xPos,yPos)にセル内容cを設定する
setCell:: Matrix -> Int -> Int -> Cell -> Matrix
setCell (Matrix mat w h) xPos yPos c
      | ( xPos < w ) && (yPos < h) = Matrix (setCell' mat xPos yPos c) w h

-- [[Cell]]型の盤面の指定した座標(xPos,yPos)にセル内容cを設定する
setCell' :: [[Cell]] -> Int -> Int -> Cell -> [[Cell]]
setCell' (x:xs) xPos yPos c
    | xPos == 0 = setCellY x yPos c : xs
    | otherwise = x:setCell' xs (xPos-1) yPos c

-- Cellの列の指定した座標(yPos)にセル内容cを設定する
setCellY :: [Cell] -> Int -> Cell -> [Cell]
setCellY (x:xs) yPos c
    | yPos == 0 = c:xs
    | otherwise = x:setCellY xs (yPos-1) c

-- 以下のtmpは、以下が通らなかったための苦肉の策。fregeのバグ?
-- fillWater m0 = foldl (\mat (x,y)-> fillWaterCell mat x y) m0 (cells m0)
tmp:: Matrix -> (Int, Int) -> Matrix
tmp mat (x,y) = fillWaterCell mat x y

-- 盤面に水を満たす
fillWater :: Matrix -> Matrix
fillWater m0 = foldl tmp m0 (cells m0)
  where
    cells (Matrix _ width height) = do
        y <- [0.. height-1]
        x <- [0.. width-1]
        return (x, y)

fillWaterCell mat x y
    | isKeepWater mat x y && getCell mat x y == Empty = setCell mat x y Water
    | otherwise = mat

-- 指定した座標x,yは水を保持できるか？
isKeepWater :: Matrix -> Int -> Int -> Bool
isKeepWater mat x y
    | hereOK mat x y = true
    | leftOK mat x y && bottomOK mat x y && rightThroughOK mat x y = true
    | otherwise = false
    where
        ok x = (x == Wall) || (x == Water)
        hereOK mat x y = ok $ getCell mat x y
        leftOK mat x y = ok $ getCell mat (x-1) y
        bottomOK mat x y = ok $ getCell mat x (y-1)
        rightOK mat x y = ok $ getCell mat (x+1) y
        rightThroughOK mat x y
            | bottomOK mat x y && rightOK mat x y = true
            | bottomOK mat x y && rightThroughOK mat (x+1) y = true
            | otherwise = false
           
-- 水の個数を返す
countAllWater :: Matrix -> Int
countAllWater (Matrix m _ _) = foldr ((+) . countAllWaterY) 0 m
    where
        countAllWaterY [] = 0
        countAllWaterY (Water:xs) = 1 + countAllWaterY xs
        countAllWaterY (x:xs) = 0 + countAllWaterY xs

-- x.atoiはHaskellのread xと等価(xがintとして解釈可能である文字列の場合)。
test :: String -> String -> Bool
test dat expected = expected.atoi == (countAllWater $ fillWater (toMatrix dat))

main :: [String] -> IO ()
main _ = do
   println $ test "83141310145169154671122" "24" {-0-}
   println $ test "923111128" "45" {-1-}
   println $ test "923101128" "1" {-2-}
   println $ test "903111128" "9" {-3-}
   println $ test "3" "0" {-4-}
   println $ test "31" "0" {-5-}
   println $ test "412" "1" {-6-}
   println $ test "3124" "3" {-7-}
   println $ test "11111" "0" {-8-}
   println $ test "222111" "0" {-9-}
   println $ test "335544" "0" {-10-}
   println $ test "1223455321" "0" {-11-}
   println $ test "000" "0" {-12-}
   println $ test "000100020003121" "1" {-13-}
   println $ test "1213141516171819181716151413121" "56" {-14-}
   println $ test "712131415161718191817161514131216" "117" {-15-}
   println $ test "712131405161718191817161514031216" "64" {-16-}
   println $ test "03205301204342100" "1" {-17-}
   println $ test "0912830485711120342" "18" {-18-}
   println $ test "1113241120998943327631001" "20" {-19-}
   println $ test "7688167781598943035023813337019904732" "41" {-20-}
   println $ test "2032075902729233234129146823006063388" "79" {-21-}
   println $ test "8323636570846582397534533" "44" {-22-}
   println $ test "2142555257761672319599209190604843" "41" {-23-}
   println $ test "06424633785085474133925235" "51" {-24-}
   println $ test "503144400846933212134" "21" {-25-}
   println $ test "1204706243676306476295999864" "21" {-26-}
   println $ test "050527640248767717738306306596466224" "29" {-27-}
   println $ test "5926294098216193922825" "65" {-28-}
   println $ test "655589141599534035" "29" {-29-}
   println $ test "7411279689677738" "34" {-30-}
   println $ test "268131111165754619136819109839402" "102" {-31-}
```
