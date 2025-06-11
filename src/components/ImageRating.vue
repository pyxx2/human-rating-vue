<template>
  <div class="image-rating-container">
    <div class="header-container">
      <h1>{{ texts.title }}</h1>
      <button @click="toggleLanguage" class="language-toggle-btn">
        {{ currentLanguage === 'en' ? '中文' : 'English' }}
      </button>
    </div>

    <div v-if="loading" class="loading">{{ texts.loading }}</div>

    <div v-else-if="imagePair" class="content">
      <div class="images-container">
        <div class="image-wrapper">
          <div class="image-title">{{ texts.groundTruth }}</div>
          <img :src="imagePair.image1_url" alt="图片1" class="image" />
        </div>
        <div class="image-wrapper">
          <div class="image-title">
           {{ texts.renderedImage }}<span v-if="imagePair.model" style="color: black;">({{ imagePair.model }})</span>
          </div>
          <img :src="imagePair.image2_url" alt="图片2" class="image" />
        </div>
      </div>

      <div class="rating-section">
        <div class="rating-header">
          <button @click="showEvaluationRules = true" class="evaluation-rules-btn">
            {{ texts.evaluationRules }}
          </button>
          <h3>{{ texts.ratingInstruction }}</h3>
          
          <!-- 显示当前分数（如果是重新评分模式） -->
          <div v-if="isReRatingMode" class="current-score">
            {{ texts.currentScore }}: {{ imagePair.current_score }}
          </div>
        </div>
        <div class="score-buttons">
          <button 
            v-for="score in 10" 
            :key="score"
            @click="selectScore(score)"
            class="score-btn"
            :class="{ 
              'selected-score-btn': selectedScore === score,
              'current-score-btn': isReRatingMode && score === imagePair.current_score 
            }"
          >
            {{ score }}
          </button>
        </div>
      </div>

      <div class="navigation-buttons">
        <button 
          @click="loadPreviousRatedPair" 
          class="nav-btn previous-btn" 
          :disabled="loading || isReRatingMode"
        >
          {{ texts.previous }}
        </button>
        
        <button 
          @click="loadNewImagePair" 
          class="nav-btn next-btn" 
          :disabled="loading"
        >
          {{ isReRatingMode ? texts.backToNew : texts.next }}
        </button>

        <button 
          @click="submitRating" 
          class="nav-btn submit-btn" 
          :disabled="!selectedScore || submitting"
        >
          {{ submitting ? texts.submitting : texts.submit }}
        </button>
      </div>
    </div>

    <!-- 评估规则弹窗 -->
    <div v-if="showEvaluationRules" class="modal-overlay" @click="showEvaluationRules = false">
      <div class="modal-content" @click.stop>
        <div class="modal-header">
          <h2>{{ texts.evaluationRules }}</h2>
          <button @click="showEvaluationRules = false" class="close-btn">&times;</button>
        </div>
        <div class="modal-body">
          <table class="rules-table">
            <thead>
              <tr>
                <th>{{ texts.scoreRange }}</th>
                <th>{{ texts.description }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="rule in texts.rules" :key="rule.score">
                <td>{{ rule.score }}</td>
                <td>{{ rule.description }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';

export default {
  name: 'ImageRating',
  data() {
    return {
      imagePair: null,
      loading: false,
      submitting: false,
      apiUrl: 'http://localhost:8000',
      showEvaluationRules: false,
      currentLanguage: 'en',
      isReRatingMode: false, // 是否处于重新评分模式
      selectedScore: null // 当前选中的分数
    };
  },
  computed: {
    texts() {
      const translations = {
        en: {
          title: 'Image Similarity Scoring System',
          groundTruth: 'ground_truth',
          renderedImage: 'rendered_image',
          evaluationRules: 'Evaluation Rules',
          ratingInstruction: 'The left is the ground truth, and what is similary level of the diagra on the right? Please rate this set of images (1 to 10).',
          next: 'Next',
          previous: 'Previous (Re-rate)',
          backToNew: 'Back to New Images',
          submit: 'Submit',
          submitting: 'Submitting...',
          loading: 'loading...',
          currentScore: 'Current Score',
          scoreRange: 'Score Range',
          description: 'Description',
          rules: [
            {
              score: '10',
              description: 'Almost completely identical, with only very minor differences (such as color/font)'
            },
            {
              score: '9',
              description: 'All major structures and semantics are consistent, layout arrangement may be different (horizontal layout becomes vertical layout, elements originally on the left go to the right.)'
            },
            {
              score: '7-8',
              description: 'The structure (boxes, connection relationships) are all there, but the content in the structure, such as text, is missing or incorrect.'
            },
            {
              score: '5-6',
              description: 'There are many errors or omissions in the image, such as missing structures (boxes), missing or confused connection relationships'
            },
            {
              score: '3-4',
              description: 'The image and the original image look inconsistent in overall structure, with only some similar elements (such as boxes)'
            },
            {
              score: '1-2',
              description: 'The image has serious errors and is obviously not the same type of image at first glance. Or even if it is the same type of image, the structure, text content, and connection relationships are very mismatched.'
            }
          ]
        },
        zh: {
          title: '图像相似度评分系统',
          groundTruth: '标准图',
          renderedImage: '生成图',
          evaluationRules: '评分规则',
          ratingInstruction: '左边是标准图，右边图像的相似度如何？请为这组图像评分（1到10分）。',
          next: '下一组',
          previous: '上一组（重新评分）',
          backToNew: '返回新图片',
          submit: '提交',
          submitting: '提交中...',
          loading: '加载中...',
          currentScore: '当前分数',
          scoreRange: '分数范围',
          description: '说明',
          rules: [
            {
              score: '10分',
              description: '几乎完全一致，只有非常细微的差异（比如颜色/字体）'
            },
            {
              score: '9分',
              description: '所有主要结构和语义一致，布局排布可能不一样（水平布局变成垂直布局、元素本来在左边去了右边。）'
            },
            {
              score: '7~8分',
              description: '结构（框，连接关系）都在，但是结构中的内容，比如文本有缺失，错误。'
            },
            {
              score: '5~6分',
              description: '图中出现较多错误或遗漏，比如结构（框）有缺失、连接关系缺少或混乱'
            },
            {
              score: '3~4分',
              description: '图像和原图看上去整体结构不一致，只有部分元素类似（比如框）'
            },
            {
              score: '1~2分',
              description: '图像严重错误，一眼看上去就不是一个类型的图。或者就算是一个类型的图，结构、文本内容、连接关系很不匹配。'
            }
          ]
        }
      };
      return translations[this.currentLanguage];
    }
  },
  mounted() {
    this.loadImagePair();
  },
  methods: {
    toggleLanguage() {
      this.currentLanguage = this.currentLanguage === 'en' ? 'zh' : 'en';
    },
    
    // 选择分数（不提交）
    selectScore(score) {
      this.selectedScore = score;
    },
    
    async loadImagePair() {
      this.loading = true;
      this.isReRatingMode = false;
      this.selectedScore = null; // 重置选中的分数
      try {
        const response = await axios.get(`${this.apiUrl}/api/image-pair`);
        this.imagePair = response.data;
        console.log('New image pair loaded:', this.imagePair);
      } catch (error) {
        console.error('加载图片失败:', error);
        if (error.response?.status === 404) {
          alert('所有图像都已评分完成！');
        } else {
          alert('加载图片失败，请检查后端服务');
        }
      } finally {
        this.loading = false;
      }
    },

    async loadPreviousRatedPair() {
      this.loading = true;
      this.selectedScore = null; // 重置选中的分数
      try {
        const response = await axios.get(`${this.apiUrl}/api/previous-rated-pair`);
        this.imagePair = response.data;
        this.isReRatingMode = true;
        this.selectedScore = this.imagePair.current_score; // 设置为当前已有的分数
        console.log('Previous rated pair loaded:', this.imagePair);
      } catch (error) {
        console.error('加载上一组图片失败:', error);
        if (error.response?.status === 404) {
          alert('没有找到已评分的记录！');
        } else {
          alert('加载上一组图片失败');
        }
      } finally {
        this.loading = false;
      }
    },

    async submitRating() {
      if (!this.imagePair || !this.selectedScore) return;
      
      this.submitting = true;
      try {
        if (this.isReRatingMode) {
          // 更新已有评分
          await axios.put(`${this.apiUrl}/api/rating/${this.imagePair.rating_id}`, null, {
            params: { new_score: this.selectedScore }
          });
          alert(`评分更新成功！新分数: ${this.selectedScore}`);
          // 更新当前显示的分数
          this.imagePair.current_score = this.selectedScore;
        } else {
          // 提交新评分
          await axios.post(`${this.apiUrl}/api/rating`, {
            image1_id: this.imagePair.image1_id,
            image2_id: this.imagePair.image2_id,
            score: this.selectedScore
          });
          // alert(`评分提交成功！分数: ${this.selectedScore}`);
          // 自动加载下一组
          await this.loadImagePair();
        }
      } catch (error) {
        console.error('评分提交失败:', error);
        if (error.response?.status === 404) {
          if (this.isReRatingMode) {
            alert('评分记录不存在！');
          } else {
            alert('所有图像都已评分完成！');
          }
        } else {
          alert('评分提交失败，请重试');
        }
      } finally {
        this.submitting = false;
      }
    },

    async loadNewImagePair() {
      if (this.isReRatingMode) {
        // 如果在重新评分模式，返回到新图片模式
        await this.loadImagePair();
      } else {
        // 正常加载下一组
        await this.loadImagePair();
      }
    }
  }
};
</script>

<style scoped>
 * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body, html {
            height: 100%;
            overflow: hidden;
        }

        .image-rating-container {
            width: 100vw;
            height: 100vh;
            display: flex;
            flex-direction: column;
            background: linear-gradient(to bottom, #fdfbfb, #ebedee);
            font-family: 'Segoe UI', 'Helvetica Neue', sans-serif;
            background-size: cover;
            background-position: center;
        }

        .header-container {
            display: flex;
            align-items: center;
            justify-content: center;
            position: relative;
            margin: 15px 0 10px;
            flex-shrink: 0;
        }

        h1 {
            font-size: 28px;
            color: #222;
            margin: 0;
            position: relative;
        }

        h1::after {
            content: "";
            display: block;
            width: 60px;
            height: 3px;
            margin: 8px auto 0;
            background: linear-gradient(to right, #007bff, #00c6ff);
            border-radius: 2px;
        }

        .language-toggle-btn {
            position: absolute;
            right: 20px;
            padding: 8px 16px;
            font-size: 14px;
            font-weight: 500;
            background: linear-gradient(to right, #667eea, #764ba2);
            color: white;
            border: none;
            border-radius: 20px;
            cursor: pointer;
            transition: all 0.3s ease;
            box-shadow: 0 3px 8px rgba(0, 0, 0, 0.15);
            min-width: 80px;
        }

        .language-toggle-btn:hover {
            transform: translateY(-1px);
            box-shadow: 0 4px 10px rgba(0, 0, 0, 0.2);
            opacity: 0.9;
        }

        .content {
            flex: 1;
            display: flex;
            flex-direction: column;
            min-height: 0;
        }

        .images-container {
            flex: 1;
            display: flex;
            width: 100%;
            padding: 10px 15px;
            gap: 15px;
            min-height: 0;
        }

        .image-wrapper {
            flex: 1;
            position: relative;
            border: 2px solid #ccc;
            border-radius: 12px;
            background: linear-gradient(145deg, #ffffff, #e6e6e6);
            box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            padding-top: 35px;
            min-height: 0;
        }

        .image-title {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            text-align: center;
            background-color: rgba(79, 143, 255, 0.6);
            color: #fff;
            padding: 8px 0;
            font-size: 23px;
            z-index: 10;
            border-top-left-radius: 12px;
            border-top-right-radius: 12px;
            user-select: none;
        }

        .image {
            max-width: 85%;
            max-height: 85%;
            object-fit: contain;
            border-radius: 8px;
            transition: transform 0.3s ease;
        }

        .image:hover {
            transform: scale(1.02);
        }

        .rating-section {
            padding: 15px 20px;
            text-align: center;
            flex-shrink: 0;
        }

        .rating-header {
            display: flex;
            align-items: center;
            justify-content: center;
            gap: 20px;
            margin-bottom: 15px;
            position: relative;
        }

        .evaluation-rules-btn {
            position: absolute;
            left: 0;
            padding: 8px 16px;
            font-size: 14px;
            font-weight: 500;
            background: linear-gradient(to right, #ff6b6b, #ee5a52);
            color: white;
            border: none;
            border-radius: 20px;
            cursor: pointer;
            transition: all 0.3s ease;
            box-shadow: 0 3px 8px rgba(0, 0, 0, 0.15);
        }

        .evaluation-rules-btn:hover {
            transform: translateY(-1px);
            box-shadow: 0 4px 10px rgba(0, 0, 0, 0.2);
            opacity: 0.9;
        }

        h3 {
            color: #444;
            font-weight: 500;
            font-size: 18px;
            margin: 0;
            flex: 1;
            text-align: center;
        }

        .current-score {
            position: absolute;
            right: 0;
            padding: 8px 16px;
            background: linear-gradient(to right, #28a745, #20c997);
            color: white;
            border-radius: 20px;
            font-size: 14px;
            font-weight: 500;
            box-shadow: 0 3px 8px rgba(0, 0, 0, 0.15);
        }

        .score-buttons {
            display: flex;
            justify-content: center;
            gap: 8px;
            flex-wrap: wrap;
        }

        .score-btn {
            padding: 10px 16px;
            font-size: 20px;
            font-weight: bold;
            color: white;
            border: none;
            border-radius: 20px;
            cursor: pointer;
            transition: all 0.3s ease;
            min-width: 40px;
            box-shadow: 0 3px 8px rgba(0, 0, 0, 0.15);
        }

        /* 选中效果 */
        .selected-score-btn {
            box-shadow: 0 0 0 3px rgba(0, 123, 255, 0.8);
            transform: scale(1.1);
        }

        /* 当前分数高亮 */
        .current-score-btn {
            box-shadow: 0 0 0 3px rgba(255, 215, 0, 0.6);
            transform: scale(1.1);
        }

        /* 1分 - 深绿色 */
        .score-btn:nth-child(1) {
            background: linear-gradient(to right, #a0c15a, #8fb048);
        }

        /* 2分 - 浅绿色 */
        .score-btn:nth-child(2) {
            background: linear-gradient(to right, #add633, #9bc429);
        }

        /* 3.4分 - 黄色 */
        .score-btn:nth-child(3),
        .score-btn:nth-child(4) {
            background: linear-gradient(to right, #ffd934, #f0c929);
        }

        /* 5-6分 - 橙色 */
        .score-btn:nth-child(5),
        .score-btn:nth-child(6) {
            background: linear-gradient(to right, #ffb234, #f0a029);
        }

        /* 7.8分 - 浅红色 */
        .score-btn:nth-child(7),
        .score-btn:nth-child(8) {
            background: linear-gradient(to right, #ff8c5a, #f07a48);
        }

        /* 9.10分 - 深红色 */
        .score-btn:nth-child(9),
        .score-btn:nth-child(10) {
            background: linear-gradient(to right, #ff4444, #e63939);
        }

        .score-btn:hover:not(:disabled) {
            transform: translateY(-1px);
            box-shadow: 0 4px 10px rgba(0, 0, 0, 0.2);
            filter: brightness(1.1);
        }

        .score-btn:disabled {
            opacity: 0.5;
            cursor: not-allowed;
        }

        /* 导航按钮容器 */
        .navigation-buttons {
            display: flex;
            justify-content: center;
            gap: 20px;
            margin: 15px auto 20px;
            flex-shrink: 0;
        }

        .nav-btn {
            padding: 12px 32px;
            font-size: 16px;
            font-weight: bold;
            color: white;
            border: none;
            border-radius: 25px;
            cursor: pointer;
            box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
            transition: all 0.3s ease;
            min-width: 120px;
        }

        /* 上一组按钮 - 紫色渐变 */
        .previous-btn {
            background: linear-gradient(to right, #667eea, #764ba2);
        }

        .previous-btn:hover:not(:disabled) {
            background: linear-gradient(to right, #5a6fd8, #6a4190);
            transform: translateY(-1px);
            box-shadow: 0 6px 14px rgba(0, 0, 0, 0.2);
        }

        /* 下一组按钮 - 绿色渐变 */
        .next-btn {
            background: linear-gradient(to right, #00c851, #007e33);
        }

        .next-btn:hover:not(:disabled) {
            background: linear-gradient(to right, #28a745, #218838);
            transform: translateY(-1px);
            box-shadow: 0 6px 14px rgba(0, 0, 0, 0.2);
        }

        /* 提交按钮 - 红色渐变 */
        .submit-btn {
            background: linear-gradient(to right, #dc3545, #c82333);
        }

        .submit-btn:hover:not(:disabled) {
            background: linear-gradient(to right, #c82333, #bd2130);
            transform: translateY(-1px);
            box-shadow: 0 6px 14px rgba(0, 0, 0, 0.2);
        }

        .nav-btn:disabled {
            opacity: 0.5;
            cursor: not-allowed;
        }

        .loading {
            font-size: 18px;
            color: #777;
            text-align: center;
            margin-top: 40px;
        }

        /* 弹窗样式 */
        .modal-overlay {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0, 0, 0, 0.5);
            display: flex;
            justify-content: center;
            align-items: center;
            z-index: 1000;
        }

        .modal-content {
            background: white;
            border-radius: 12px;
            box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
            max-width: 80%;
            max-height: 80%;
            overflow: hidden;
            display: flex;
            flex-direction: column;
        }

        .modal-header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            padding: 20px 25px;
            border-bottom: 1px solid #e0e0e0;
            background: linear-gradient(to right, #667eea, #764ba2);
            color: white;
        }

        .modal-header h2 {
            margin: 0;
            font-size: 20px;
            font-weight: 600;
        }

        .close-btn {
            background: none;
            border: none;
            font-size: 24px;
            color: white;
            cursor: pointer;
            padding: 0;
            width: 30px;
            height: 30px;
            display: flex;
            align-items: center;
            justify-content: center;
            border-radius: 50%;
            transition: background-color 0.3s ease;
        }

        .close-btn:hover {
            background-color: rgba(255, 255, 255, 0.2);
        }

        .modal-body {
            padding: 25px;
            overflow-y: auto;
        }

        .rules-table {
            width: 100%;
            border-collapse: collapse;
            font-size: 14px;
        }

        .rules-table th,
        .rules-table td {
            padding: 12px 15px;
            text-align: left;
            border-bottom: 1px solid #e0e0e0;
        }

        .rules-table th {
            background: linear-gradient(to right, #f8f9fa, #e9ecef);
            font-weight: 600;
            color: #495057;
            position: sticky;
            top: 0;
        }

        .rules-table tr:hover {
            background-color: #f8f9fa;
        }

        .rules-table td:first-child {
            font-weight: 600;
            color: #007bff;
            white-space: nowrap;
        }

        .rules-table td:last-child {
            line-height: 1.5;
        }
        
</style>