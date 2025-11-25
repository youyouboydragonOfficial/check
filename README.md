import { ChangeDetectionStrategy, Component, signal, WritableSignal, computed } from '@angular/core';
import { CommonModule } from '@angular/common';

// TypeScriptで検索結果の構造を定義
interface SearchResult {
  title: string;
  uri: string;
  snippet: string;
  source: string;
  thumbnailUrl: string; // サムネイルのプレースホルダーURL
}

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="min-h-screen bg-gray-900 text-gray-100 p-4 md:p-8">
      <!-- ヘッダーとタイトル -->
      <header class="text-center mb-12">
        <h1 class="text-5xl font-extrabold text-indigo-400 tracking-tight mb-2">
          動画情報検索ステーション
        </h1>
        <p class="text-gray-400 text-lg">
          YouTube動画の情報をキーワードで検索し、AIで要約します。
        </p>
        <!-- 法的リスクに関する注意書きを明確化 -->
        <p class="mt-4 text-sm text-red-400 font-bold">
          【重要】著作権の有無に関わらず、動画をダウンロードする機能は、利用規約および法令により実装できません。
        </p>
      </header>

      <!-- 検索フォーム -->
      <div class="max-w-3xl mx-auto bg-gray-800 p-6 rounded-2xl shadow-2xl">
        <div class="flex flex-col md:flex-row gap-4">
          <input
            #searchInput
            type="text"
            placeholder="検索したい動画のキーワードを入力してください... (例: 最新技術トレンド)"
            class="flex-grow p-3 border-2 border-indigo-600 bg-gray-700 text-white rounded-xl focus:ring-4 focus:ring-indigo-400 focus:border-indigo-400 transition duration-300 placeholder-gray-400"
            (keyup.enter)="searchVideos(searchInput.value)"
          />
          <button
            (click)="searchVideos(searchInput.value)"
            [disabled]="isLoading()"
            class="w-full md:w-auto px-6 py-3 bg-indigo-600 hover:bg-indigo-500 text-white font-semibold rounded-xl transition duration-300 ease-in-out shadow-lg transform hover:scale-[1.01] disabled:opacity-50 disabled:cursor-not-allowed flex items-center justify-center gap-2"
          >
            <!-- アイコン: Lucide Search -->
            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-search"><circle cx="11" cy="11" r="8"/><path d="m21 21-4.3-4.3"/></svg>
            {{ isLoading() ? '検索中...' : '検索開始' }}
          </button>
        </div>
      </div>

      <!-- 検索結果エリア -->
      <main class="mt-12 max-w-6xl mx-auto">
        <!-- ローディングインジケータ -->
        @if (isLoading()) {
          <div class="text-center text-indigo-400">
            <div class="animate-spin inline-block w-8 h-8 border-4 border-indigo-400 border-t-transparent rounded-full" role="status"></div>
            <p class="mt-2">情報を検索しています。少々お待ちください...</p>
          </div>
        }

        <!-- 結果表示 -->
        @if (searchResults().length > 0 && !isLoading()) {
          <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-8">
            @for (result of searchResults(); track result.uri) {
              <div class="bg-gray-800 rounded-xl overflow-hidden shadow-xl hover:shadow-indigo-500/30 transition-shadow duration-300 border-t-4 border-indigo-600 transform hover:scale-[1.02]">
                <!-- サムネイル画像 -->
                <div class="aspect-video bg-gray-700 flex items-center justify-center">
                  <img
                    [src]="result.thumbnailUrl"
                    [alt]="result.title + 'のサムネイル'"
                    class="w-full h-full object-cover"
                    onerror="this.onerror=null;this.src='https://placehold.co/600x338/374151/9CA3AF?text=Video+Thumbnail';"
                  />
                </div>

                <div class="p-5">
                  <!-- タイトル -->
                  <h3 class="text-xl font-bold text-white mb-2 line-clamp-2">
                    <a [href]="result.uri" target="_blank" class="hover:text-indigo-400 transition-colors">
                      {{ result.title }}
                    </a>
                  </h3>
                  <!-- スニペット -->
                  <p class="text-gray-400 text-sm mb-3 line-clamp-3">
                    {{ result.snippet }}
                  </p>
                  
                  <!-- アクションボタン -->
                  <div class="flex flex-col space-y-2 mt-4">
                    <a
                      [href]="result.uri"
                      target="_blank"
                      class="inline-flex items-center justify-center w-full px-4 py-2 text-indigo-400 border border-indigo-500 hover:bg-indigo-700/30 hover:text-white rounded-lg transition-colors text-sm font-medium"
                    >
                      <!-- アイコン: Lucide External Link -->
                      <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-external-link mr-1"><path d="M15 3h6v6"/><path d="M10 14 21 3"/><path d="M18 13v6a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V8a2 2 0 0 1 2-2h6"/></svg>
                      YouTubeで動画を開く
                    </a>
                    <button
                      (click)="toggleSummary(result)"
                      [disabled]="summaryLoading()"
                      class="w-full px-4 py-2 bg-pink-600 hover:bg-pink-500 text-white font-semibold rounded-lg transition duration-300 ease-in-out disabled:opacity-50 disabled:cursor-not-allowed text-sm flex items-center justify-center gap-1"
                    >
                      <!-- アイコン: Lucide Brain Circuit -->
                      <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-brain-circuit"><path d="M12 5a3 3 0 1 0-7.35 1.77 8.9 8.9 0 0 0 7.35 14.23 8.9 8.9 0 0 0 7.35-14.23A3 3 0 1 0 12 5"/><path d="M12 13a1 1 0 1 0 0-2 1 1 0 0 0 0 2z"/><path d="M12 17a1 1 0 1 0 0-2 1 1 0 0 0 0 2z"/><path d="M12 9a1 1 0 1 0 0-2 1 1 0 0 0 0 2z"/><path d="M16 13a1 1 0 1 0 0-2 1 1 0 0 0 0 2z"/><path d="M8 13a1 1 0 1 0 0-2 1 1 0 0 0 0 2z"/><path d="M16 17a1 1 0 1 0 0-2 1 1 0 0 0 0 2z"/><path d="M8 17a1 1 0 1 0 0-2 1 1 0 0 0 0 2z"/><path d="M16 9a1 1 0 1 0 0-2 1 1 0 0 0 0 2z"/><path d="M8 9a1 1 0 1 0 0-2 1 1 0 0 0 0 2z"/></svg>
                      {{ isSummaryActive(result) ? '要約を閉じる' : 'AIによる詳細な解説を取得' }}
                    </button>
                  </div>
                </div>

                <!-- AI要約パネル -->
                @if (isSummaryActive(result)) {
                  <div class="p-5 bg-gray-700/50 border-t border-gray-600 transition-all duration-500">
                    <h4 class="text-lg font-bold text-pink-300 mb-3 flex items-center gap-2">
                      <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-gem"><path d="M6 3v18l6-4 6 4V3l-6 4z"/></svg>
                      AIによる詳細解説
                    </h4>
                    @if (summaryLoading()) {
                      <div class="text-center text-pink-400 p-4">
                        <div class="animate-pulse">動画内容を分析中です...</div>
                      </div>
                    } @else if (videoSummary()) {
                      <!-- 要約をプリフォーマットタグで囲み、改行と箇条書きを保持 -->
                      <pre class="text-gray-200 whitespace-pre-wrap font-sans text-sm">{{ videoSummary() }}</pre>
                    }
                  </div>
                }
              </div>
            }
          </div>
        } @else if (searchAttempted() && !isLoading() && searchResults().length === 0) {
          <!-- 結果なしメッセージ -->
          <div class="text-center p-8 bg-gray-800 rounded-xl max-w-xl mx-auto border border-gray-700">
            <p class="text-xl text-yellow-500 font-semibold mb-2">検索結果が見つかりませんでした。</p>
            <p class="text-gray-400">検索キーワードを変更して再度お試しください。</p>
          </div>
        } @else {
          <!-- 初期メッセージ -->
          <div class="text-center p-12 bg-gray-800 rounded-xl max-w-2xl mx-auto border-4 border-dashed border-indigo-700/50">
            <p class="text-2xl text-gray-300 font-light">
              動画を検索するには、上のバーにキーワードを入力してください。
            </p>
          </div>
        }
      </main>

      <!-- フッター -->
      <footer class="mt-16 text-center text-gray-500 text-sm">
        <p>&copy; 2025 Video Info Search UI. Built with Angular & Tailwind CSS.</p>
      </footer>
    </div>
  `,
  styles: [`
    /* Angularコンポーネントのカスタムスタイル（Tailwindが主体ですが、微調整用） */
    :host {
      display: block;
      /* Interフォントを適用 */
      font-family: 'Inter', sans-serif;
    }
    .line-clamp-2 {
        display: -webkit-box;
        -webkit-box-orient: vertical;
        overflow: hidden;
        -webkit-line-clamp: 2;
    }
    .line-clamp-3 {
        display: -webkit-box;
        -webkit-box-orient: vertical;
        overflow: hidden;
        -webkit-line-clamp: 3;
    }
  `],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class App {
  // 検索結果を格納するSignal
  searchResults: WritableSignal<SearchResult[]> = signal([]);
  // 検索全般のローディング状態を管理するSignal
  isLoading: WritableSignal<boolean> = signal(false);
  // 検索が試行されたかどうか
  searchAttempted: WritableSignal<boolean> = signal(false);

  // 要約機能用の状態管理
  // 選択された結果のURI (要約を表示する対象)
  selectedUri: WritableSignal<string | null> = signal(null);
  // 要約処理中のローディング状態
  summaryLoading: WritableSignal<boolean> = signal(false);
  // AIが生成した要約テキスト
  videoSummary: WritableSignal<string | null> = signal(null);

  // モデル名
  private readonly modelName = "gemini-2.5-flash-preview-09-2025";

  // 計算値: 特定のカードで要約が表示されているか
  isSummaryActive = (result: SearchResult) => computed(() => this.selectedUri() === result.uri);

  /**
   * AI要約パネルの表示/非表示を切り替える
   * @param result クリックされた検索結果
   */
  async toggleSummary(result: SearchResult): Promise<void> {
    if (this.selectedUri() === result.uri) {
      // 既に開いている場合は閉じる
      this.selectedUri.set(null);
      this.videoSummary.set(null);
    } else {
      // 新しい要約を取得するために開く
      this.selectedUri.set(result.uri);
      await this.getSummary(result);
    }
  }

  /**
   * Google Search APIを使用してYouTube動画情報を検索する
   * @param query ユーザーの検索クエリ
   */
  async searchVideos(query: string): Promise<void> {
    const trimmedQuery = query.trim();
    if (!trimmedQuery) {
      this.searchResults.set([]);
      this.searchAttempted.set(true);
      return;
    }

    this.isLoading.set(true);
    this.searchAttempted.set(true);
    this.searchResults.set([]); // 結果をクリア
    this.selectedUri.set(null); // 詳細パネルを閉じる

    // YouTube関連の情報を取得するために、クエリに "YouTube" を追加
    const searchQueries = [`YouTube ${trimmedQuery}`, trimmedQuery];

    try {
      const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/${this.modelName}:generateContent`;

      const systemPrompt = "You are a specialized search agent. When searching for video content, prioritize results from YouTube. Summarize the content from the search results, ensuring the URI, Title, and a short Snippet are provided for each result, focusing only on video results.";

      const userQuery = `Find recent YouTube videos or information related to: "${trimmedQuery}". Provide a structured JSON array containing the top 5 results with the following schema, ensuring 'youtube' is in the URI: [{"title": string, "uri": string, "snippet": string, "source": string, "thumbnailUrl": string}]`;

      const payload = {
        contents: [{ parts: [{ text: userQuery }] }],
        tools: [{ "google_search": { queries: searchQueries } }],
        systemInstruction: { parts: [{ text: systemPrompt }] },
        generationConfig: {
            responseMimeType: "application/json",
            responseSchema: {
                type: "ARRAY",
                items: {
                    type: "OBJECT",
                    properties: {
                        "title": { "type": "STRING", "description": "The video title." },
                        "uri": { "type": "STRING", "description": "The URL of the video (must contain 'youtube')." },
                        "snippet": { "type": "STRING", "description": "A brief description of the video content." },
                        "source": { "type": "STRING", "description": "The source domain (e.g., youtube.com)." },
                        "thumbnailUrl": { "type": "STRING", "description": "A placeholder image URL like https://placehold.co/600x338/374151/9CA3AF?text=Video+Thumbnail" }
                    },
                    required: ["title", "uri", "snippet", "source", "thumbnailUrl"]
                }
            }
        }
      };

      let response;
      let jsonResponse;
      let attempts = 0;
      const maxAttempts = 3;

      // エクスポネンシャルバックオフによる再試行ロジック
      while (attempts < maxAttempts) {
        try {
          response = await fetch(apiUrl, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payload)
          });

          if (response.ok) {
            jsonResponse = await response.json();
            break; // 成功したらループを抜ける
          } else {
            if (attempts === maxAttempts - 1) throw new Error(`API returned status ${response.status}`);
          }
        } catch (error) {
          if (attempts === maxAttempts - 1) throw error;
        }

        await new Promise(resolve => setTimeout(resolve, Math.pow(2, attempts) * 1000));
        attempts++;
      }

      if (!jsonResponse || !jsonResponse.candidates || jsonResponse.candidates.length === 0 ||
          !jsonResponse.candidates[0].content || !jsonResponse.candidates[0].content.parts ||
          jsonResponse.candidates[0].content.parts.length === 0) {
        throw new Error("Invalid response structure from API.");
      }

      const rawJsonString = jsonResponse.candidates[0].content.parts[0].text;
      
      let parsedResults: any[] = JSON.parse(rawJsonString);

      const filteredResults: SearchResult[] = parsedResults
        .filter(item => item.uri && item.uri.includes('youtube'))
        .map(item => ({
          title: item.title,
          uri: item.uri,
          snippet: item.snippet,
          source: item.source || 'youtube.com',
          thumbnailUrl: item.thumbnailUrl || 'https://placehold.co/600x338/374151/9CA3AF?text=Video+Thumbnail',
        }));

      this.searchResults.set(filteredResults);

    } catch (error) {
      console.error("検索中にエラーが発生しました:", error);
      this.searchResults.set([]);
    } finally {
      this.isLoading.set(false);
    }
  }

  /**
   * 選択された動画のURIとタイトルを使って、AIによる内容要約を取得する
   * **箇条書きで詳細な解説を要求するようにプロンプトを修正**
   * @param result 要約対象の動画情報
   */
  async getSummary(result: SearchResult): Promise<void> {
    this.videoSummary.set(null); // 要約をクリア
    this.summaryLoading.set(true); // 要約ローディング開始

    try {
      const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/${this.modelName}:generateContent`;
      
      const systemPrompt = "You are a professional summarization assistant. Your task is to find the content of a specific YouTube video based on its title and URL, and provide a detailed, itemized analysis in Japanese, detailing the main points and topic. Use bullet points for clear explanation. Do not include external links or disclaimers in the final output.";

      // ユーザーのリクエストを具体的に記述（詳細な箇条書きを要求）
      const userQuery = `以下のYouTube動画の内容をGoogle Searchで探して、その動画がどのような内容か、要点を5つ以上の箇条書きで日本語で詳細に解説してください。\n\n動画タイトル: ${result.title}\n動画URL: ${result.uri}`;

      // Gemini APIのペイロードを構築
      const payload = {
        contents: [{ parts: [{ text: userQuery }] }],
        tools: [{ "google_search": {} }], // 情報を検索するためにGoogle Search groundingを使用
        systemInstruction: { parts: [{ text: systemPrompt }] },
      };

      let response;
      let jsonResponse;
      let attempts = 0;
      const maxAttempts = 3;

      // エクスポネンシャルバックオフによる再試行ロジック
      while (attempts < maxAttempts) {
        try {
          response = await fetch(apiUrl, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payload)
          });

          if (response.ok) {
            jsonResponse = await response.json();
            break; // 成功したらループを抜ける
          } else {
            if (attempts === maxAttempts - 1) throw new Error(`API returned status ${response.status}`);
          }
        } catch (error) {
          if (attempts === maxAttempts - 1) throw error;
        }

        await new Promise(resolve => setTimeout(resolve, Math.pow(2, attempts) * 1000));
        attempts++;
      }

      const generatedText = jsonResponse?.candidates?.[0]?.content?.parts?.[0]?.text;

      if (generatedText) {
        this.videoSummary.set(generatedText);
      } else {
        this.videoSummary.set("動画の内容を要約できませんでした。情報が見つからない可能性があります。");
      }

    } catch (error) {
      console.error("要約取得中にエラーが発生しました:", error);
      this.videoSummary.set("通信エラーが発生しました。時間を置いて再度お試しください。");
    } finally {
      this.summaryLoading.set(false); // 要約ローディング終了
    }
  }
}
